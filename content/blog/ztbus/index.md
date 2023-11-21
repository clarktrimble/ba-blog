---
title: ZTBus
description: Programmatic Elasticsearch Aggregations with Golang
date: 2023-11-15
tags:
  - golang
  - database
layout: layouts/post.njk
---

{% image "./ztbus.png", "An Electric Bus in Switzerland" %}

Today we'll investigate programmatic access to Elasticsearch aggregations via Golang.

## Programmatic You Say?

Yeah!  While pleasing graphics are indeed pleasing and often quite useful for us humans, the power of aggregated data can be equally useful as intellegence to inform downstream automation.
Classic examples include finding top-talkers and long-duration responders.
Another significant use case is the conversion of data captured at the time of an event (logs), to a time-series (stats) by something as simple as a count aggreation.

## ZTBus Dataset

There's nothing particular about the ZTBus dataset with respect to this post.
We need some grist for the aggregation mill and it's publicly [available](https://www.research-collection.ethz.ch/handle/20.500.11850/626723).
The data is per-second and mostly numerical relating to such things as power consumtion, individual wheel velocities/steering angles, and brake pad pressure.
There's a recent [article](https://www.nature.com/articles/s41597-023-02600-6) with Nature for more insight into the data.

We'll be looking at `odometry_vehicleSpeed` for both bus `B183` and `B208`.

## Elasticsearch

### In the Lab

Elasticsearch was a many-headed beast when I first started hanging around with it years ago, and it's only grown.
Our focus today will be to stand one up in the lab, plomp some data into an index and run an aggregation.

Following the excellent standalone [instructions](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html) (thanks Elastic!):

```bash
docker run --name es01 --net elastic -p 9200:9200 -it -m 1GB docker.elastic.co/elasticsearch/elasticsearch:8.11.0
```

Splat! Looks like I need to tweak a kernel setting first:

```bash
sudo vi /etc/sysctl.conf

### adding "vm.max_map_count=262144"

sudo sysctl -p
docker rm es01
docker run --name es01 --net elastic -p 9200:9200 -it -m 1GB docker.elastic.co/elasticsearch/elasticsearch:8.11.0
```

That did the trick!  Now the above command starts ES and spits out some credentials, which I put in a safe place.

Let's check that its responding on 9200:

```bash
$ export ELASTIC_PASSWORD=top*secret
$ curl -k -u elastic:$ELASTIC_PASSWORD https://localhost:9200
{
  "name" : "b1c736e00684",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "7wgOc-FLRYWZRi7cgNZy3g",
  "version" : {
    "number" : "8.11.0",
    "..." : "...",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

Woot!

At some point my terminal was closed and I was able to restart the container in the background with:

```bash
docker start es01
```

Next time, I'll try starting the container in the background and reset pw with exec:

```bash
docker run --name es01 --net elastic -p 9200:9200 -d -m 1GB docker.elastic.co/elasticsearch/elasticsearch:8.11.0
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

### The End

For the Elasticsearch setup that is.
From here we'll start shoveling data into an index; no table setup or anything.
ES just figures it out for us.

In real life however, this would not be a good approach for a project expected to scale.
Many configurables lie just below the surface!

## Golang

We'll take a quick look at parsing the csv and inserting into ES.
Then, linger a bit over the aggregation.

The code is available for reference on [GitHub](https://github.com/clarktrimble/ztbus).

### Parse

As much as I love Golang, I have to admit it's a slog to take apart a csv file.
The type infos have to come from somewhere though.
We'll take a look at a few high(low?)-lights and scamper onward.

The struct we'll be using to soak up datums:

```go
// ztbus.go
type ZtBusCols struct {
  Len            int
  ColIdx         map[string]int
  BusId          string
  Ts             []time.Time
  VehicleSpeed   []float64
}
```

Record by record action:

```go
  // excerpted from New(fn string) (ztb *ZtBusCols, err error) in ztbus.go
  rdr := csv.NewReader(file)

  record, err := rdr.Read() // header
  if err != nil {
    return
  }
  for i, field := range record {
    ztb.ColIdx[field] = i // column names from header
  }

  for {
    record, err = rdr.Read() // records
    if err == io.EOF {
      return ztbCols, nil
    }
    if err != nil {
      return
    }

    err = ztb.appendRecord(record)
    if err != nil {
      return
    }
  }
```

`ColIdx` lets us refer to columns in the header by name rather than by number, yay! 

This can be seen, in `parseFloat` which is called from `appendRecord` above:

```go
// ztbus.go
func (ztb *ZtBusCols) parseFloat(field string, record []string) (val float64, err error) {

  val, err = strconv.ParseFloat(record[ztb.ColIdx[field]], 64)
  if err != nil {
    err = errors.Wrapf(err, "failed to parse %s", field)
    return
  }

  if val == -0 {
    val = 0
  }
  if math.IsNaN(val) {
    val = 0
  }

  return
}
```

Definitely banging the rocks together!
But gets the job done :)

I wonder about the merits of [gocsv](https://github.com/gocarina/gocsv) which offers marshalling into a struct.
But I'll call your attention to the executive decisions about `-0` and `IsNan` in `parseFloat`.
I've also noticed other issues with `itcs_busRoute` and `itcs_numberOfPassengers` where the data might need to be patched up by way of gratuitous hax.

In any case the code is a simple as it is crude, unlikely to offer surprise, and hopefully soon forgotten.

### Insert

We'll be using Elastic's restful "[Index API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html)", which provides for creating documents in an index one at a time.
The "[Bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html)" would offer better performance and I'm confident ES has at least three even more performant options lurking within thier copious documentation.
We just want to demonstrate programmatic aggregation and can have a quick carrot juice or something while the machines get on with it.

// Todo: not datascience disclaimer above!!

Here's the heart of the matter:

```go
func (es *Elastic) Insert(ctx context.Context, doc any) (err error) {
  result := esResult{}

  path := fmt.Sprintf(docPath, es.Idx)
  err = es.Client.SendObject(ctx, "POST", path, doc, &result)
  if err != nil {
    return
  }

  if result.Result != "created" {
    err = errors.Errorf("unexpected result from es: %#v", result.Result)
  }
  return
}
```

Where `Client` is a fairly vanilla restful json-api http client with support for marshalling requests and unmarshallig responses.

Without further ado, let's load some records:

// Todo: link main here and for aggregate pls
// Todo: this is loadl not load-ztb!!

```bash
$ bin/load-jsonl < load-B183.log
93285 records inserted in 325.39 seconds
```

Oof, that's like over 5 minutes.  Yeah, Bulk ftw?


### Aggregate

Now that we've got some data indexed, we'll take a look at an aggregation.
Let's say, .. avgerage speed over some interval for each bus.
Even this can be a mildly daunting when confronted by the copius aggregation [documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html) over at Elastic.
I've always felt like these could do more to describe how the various aggs can be combined.
A cool trick is to use a Kibana dashboard to cook up something that's close and go from there on the programmatic side.

#### Kibana Aside

First of all, I'm a big fan.
I've troubleshot many an issue via Kibana's "Discovory" mode, which provides a filtered view of the data.

Here, we're going to look at "Dashboard" mode as a means of getting a start on an aggreation query.
You might want to skip over this section if your already cool with the mysteries of putting these together.


Let's get started by generating a token and starting Kibana up in a container:

```bash
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana

docker run --name kib01 --net elastic -p 5601:5601 docker.elastic.co/kibana/kibana:8.11.1
```

Point a browser to Kibana, plug in the enrollement token, and login as "elastic" user from above.

Be sure to create a "Data view" first if you haven't already done so.
[Instructions](https://www.elastic.co/guide/en/kibana/current/data-views.html), or in bread crumbs:

**3bar (top-left button) > Discover > "Data View Menu" (just below 3bar) > Create a data view**

- Name: something memorable
- Index pattern: something that matches just the index(es) of interest
- Timestamp field: ts
- and Save

Now for the agg!:

**3bar > Dashboard > Create a Dashboard > Add Panel > Aggregation based > Data table**

Choose the Data view from above and monkey with the agg, for example:
 - set the date range so that Count metric is not 0
 - change metric to Average and a field of interest
 - add a Split Rows bucket with Date Histogram and field "ts"
 - Update

And, with some luck, you'll get see a table with data!

Are you still with me?
One last step to get the aggregation query ...

**Inspect > View: Request > Request > Copy to clipboard**

```json
{
  "aggs": {
    "2": {
      "date_histogram": {
        "field": "ts",
        "calendar_interval": "1h",
        "time_zone": "America/Chicago",
        "min_doc_count": 1
      },
      "aggs": {
        "1": {
          "avg": {
            "field": "vehicle_speed"
          }
        }
      }
    }
  },
  "size": 0,
  "query": {
    "bool": {
      "must": [],
      "filter": [
        {
          "range": {
            "ts": {
              "format": "strict_date_optional_time",
              "gte": "2022-09-21T14:00:00.000Z",
              "lte": "2022-09-21T21:00:00.000Z"
            }
          }
        }
      ],
    }
  }
}
```

Voila!  Kibana has got us off to a good start in the mildly arcane world of Elastic aggregation querys :)
On with the show. Golang, golang, golang ..

## Programmatic Aggregation

### Templating

For programmatic aggs, we'll need a way to generate aggregation queries on the fly for variable date-ranges etc.
Hmmm, perhaps a template will do the job?:

```yml
---
aggs:
  outer:
    date_histogram:
      field: "ts"
      fixed_interval: "\{\{ .interval }}"
      time_zone: "UTC"
      min_doc_count: 1
    aggs:
      middle:
        terms:
          field: "bus_id.keyword"
        aggs:
          inner:
            avg:
              field: "vehicle_speed"
query:
  bool:
    filter:
      - range:
          "ts":
            gte: "\{\{ .bgn }}"
            lte: "\{\{ .end }}"
size: 0
```
(pardon the extra backslashes while I figure out how to drive markdown-it)

The query will:
 - filter on a date range
 - bucket the data by interval
 - bucket the interval buckets by bus_id
 - and finally get the average speed for a bus in an interval

Note the presense of the keyword ".keyword".
Recall mention of "configurables below the surface"?
When the data was thrown at an unconfigured index, Elastic decided to create the "bus_id" field as type `text`.
Which is not availble for term aggregation.
Luckily for us, it also created an nearly identical sub-field "keyword" that is up for term aggs.

The template is in yaml rather than json to improve readability for complex queries.
Compare this three-layer one the the two-layer shown in json above.

To pull this off, first the template is rendered and then converted to json:

```go
// template.go
func (tmpl *Template) RenderJson(name string, data interface{}) (out []byte, err error) {

  buf := &bytes.Buffer{}
  err = tmpl.render(name, data, buf)
  if err != nil {
    return
  }

  out, err = yaml.YAMLToJSON(buf.Bytes())
  return
}

func (tmpl *Template) render(name string, data interface{}, writer io.Writer) (err error) {

  if tmpl.Tmpl == nil {
    err = errors.Errorf("no templates loaded for: %#v", tmpl)
    return
  }

  err = tmpl.Tmpl.ExecuteTemplate(writer, fmt.Sprintf("%s.%s", name, tmpl.Suffix), data)
  return
}
```

OK, why not just marshal a struct per usual??

Have a closer look at the structure above.
"outer", "middle", and "inner" are values rather than field names.
This plays hell with Marshal/Unmarshal.

### Road Test

There's a [dump-query](https://github.com/clarktrimble/ztbus/blob/main/cmd/dump-query/main.go) utility in the ztbus project that dumps the rendered query as well as the response body from Elastic.

// Todo: link to dump!


### Wrangling Response

At this point we've got the programmatic aggregation query sorted, now to handle the response.
In the home stretch!

Here's a chunk of what we get back:

```json
    "aggregations": {
      "outer": {
        "buckets": [
          {
            "key_as_string": "2022-09-21T08:00:00.000Z",
            "key": 1663747200000,
            "doc_count": 600,
            "middle": {
              "doc_count_error_upper_bound": 0,
              "sum_other_doc_count": 0,
              "buckets": [
                {
                  "key": "B183",
                  "doc_count": 300,
                  "inner": {
                    "value": 0.99
                  }
                },
                {
                  "key": "B208",
                  "doc_count": 300,
                  "inner": {
                    "value": 4.93
                  }
                }
              ]
            }
          },
```

Ouch! generally and we have the pesky value-as-json-key thing again.

[gjson](https://github.com/tidwall/gjson) to the rescue:

```go
func (svc *Svc) AvgSpeed(ctx context.Context, data map[string]string) (avgs ztbus.AvgSpeeds, err error) {

  query, err := svc.Repo.Query("avgspeed", data)
  if err != nil {
    return
  }

  result, err := svc.Repo.Search(ctx, query)
  if err != nil {
    return
  }

  avgs = ztbus.AvgSpeeds{}
  for _, bkt1 := range gjson.GetBytes(result, "aggregations.outer.buckets").Array() {
    for _, bkt2 := range bkt1.Get("middle.buckets").Array() {

      ts := bkt1.Get("key").Int()

      avgs = append(avgs, ztbus.AvgSpeed{
        Ts:           time.UnixMilli(ts).UTC(),
        BusId:        bkt2.Get("key").String(),
        VehicleSpeed: bkt2.Get("inner.value").Float(),
      })
    }
  }

  return
}
```

In the `for` loops we see gjson making short work of pulling the data we're after from the query result and presenting us with a tidy `ztbus.AvgSpeeds` object.

Of course, this code is irretrievably coupled to the `"avgspeed"` agg query template, but in a scope small enough for a post such this :)

## The End (Really)

We've had a whirlwind into to Elastic and aggregation to set the stage for:

 - templated aggregation queries
 - unmarshalled extraction of data from aggregation results

Both of which offer a workable, pragmatic even, approach to making the considerable power of programmatic aggregation available with Golang.

Thanks for reading!



show agg dump

gjson ftw

#### Dep Rot?

## logginss

### rando

Package "wrapping" with similar name .. read about a while back somewhere, sensible yea

Not doing datascience here abouts ..

Not using es lib .. needed for cursor??

" elastic" pkg not a general purpos lib, but specific to use case, kinda??

Link to github

While intellegence informing automation is left as an excersize to to the reader's imagination, industrial strength logging is not!

Candidates for promotion to minimod: template.



