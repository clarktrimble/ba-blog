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

### Programmatic You Say?

Aggregations are a powerful means for utilizing data both as a precursor to visualizations and as a means to inform automation.
Classic examples include finding top-talkers and long-duration responders.

So rather than Jupyter Notebooks, Matlab, or other visualization related tools, we'll be aggregating with Golang for something approaching a datacenter-ready solution.

### ZTBus Dataset

There's nothing particular about the ZTBus dataset with respect to the topic at hand.
We need some grist for the aggregation mill and it's publicly [available](https://www.research-collection.ethz.ch/handle/20.500.11850/626723).
The data is per-second and mostly numerical relating to such things as power consumption, individual wheel velocities/steering angles, brake pad pressure, and so forth.
There's a recent [article](https://www.nature.com/articles/s41597-023-02600-6) from Nature for more on the dataset.

## TL;DR

Elasticserch (ES) aggregations can be a challenge to handle from a Golang perspective.
Both the queries and the results are represented as json objects and ES's use of values, rather than field names, as keys at points in the structure throw a wrench into the usual Golang json Marshal/Unmarshal.

In this post, an approach using templates to form queries and an alternative means of parsing results is demonstrated.

Companion code is available on [GitHub](https://github.com/clarktrimble/ztbus).

## Elasticsearch

### In the Lab

Elasticsearch was a many-headed lobster-squid when I first started hanging around with it years ago, and it's only grown more heads / tentacles.
Our focus today will be to setup a development instance, plomp some data into an index and run an aggregation.

Following the excellent standalone [instructions](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html) (thanks Elastic!):

```bash
docker run --name es01 --net elastic -p 9200:9200 -it -m 1GB docker.elastic.co/elasticsearch/elasticsearch:8.11.0
```

**Splat!**

Looks like I need to tweak a kernel setting first:

```bash
sudo vi /etc/sysctl.conf
### adding "vm.max_map_count=262144"
sudo sysctl -p

docker rm es01
docker run --name es01 --net elastic -p 9200:9200 -it -m 1GB docker.elastic.co/elasticsearch/elasticsearch:8.11.0
```

That did the trick!  Now the above container starts ES and spits out some credentials, which I put in a safe place.

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

Next time, I'll try starting the container in the background to begin with and reset the password via exec:

```bash
docker run --name es01 --net elastic -p 9200:9200 -d -m 1GB docker.elastic.co/elasticsearch/elasticsearch:8.11.0
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

### The End

For the ES setup, that is.
From here, we'll start shoveling data into an index; no table setup or anything.
ES just figures it out for us.

In real life however, this would not be a good approach for a project expected to scale.
Many configurables lie just below the surface!

## Ingestion

### Parse

From ZTBus we'll focus on `odometry_vehicleSpeed` over time for buses `B183` and `B208`.

The Golang struct we'll be using to soak up datums:

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

`ColIdx` lets us refer to columns by header name rather than by number, yay! 

As seen in `parseFloat` which is called from `appendRecord` above:

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
But getting the job done :)

I wonder about the merits of [gocsv](https://github.com/gocarina/gocsv) which offers marshalling into a struct.
But I'll call your attention to the executive decisions regarding `-0` and `IsNan` made in `parseFloat`.
I've also noticed other potential issues with `itcs_busRoute` and `itcs_numberOfPassengers` where the data might need to be patched up by way of gratuitous hax.

In any case the code is a simple as it is crude, unlikely to offer surprise, and doubtless soon forgotten.

### Insert

We'll be using Elastic's restful [Index API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html), which provides for creating documents in an index one at a time.
The [Bulk API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html) would offer better performance and I'm confident ES has even more performant options lurking within their copious documentation.
We just want some indexed data and can have a quick carrot juice or something while the machines get on with it.

Here's the heart of the matter:

```go
// elastic.go
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

Where `Client` is a fairly vanilla restful json-api http client with support for marshalling requests and unmarshalling responses.

And `doc`s are pulled from the columnar representation with:

```go
// ztbus.go
func (ztb *ZtBusCols) Row(i int) *ZtBus {
  return &ZtBus{
    BusId:          ztb.BusId,
    Ts:             ztb.Ts[i],
    VehicleSpeed:   ztb.VehicleSpeed[i],
  }
}
```

Without further ado, let's load some records:

```bash
$ ZTB_SVC_DRYRUN=false bin/load-ztbus 2>&1 | tee load-B183.log
{"config":"{\"version\":\"more-agg.12.48f764f\",\"http_client\":{\"base_uri\":\"https://localhost:9200\",\"timeout\":60000000000,\"timeout_short\":10000000000,\"skip_verify\":true,\"user\":\"elastic\",\"pass\":\"--redacted--\"},\"es\":{\"es_index\":\"ztbus003\"},\"ztb_svc\":{\"dry_run\":false},\"truncate\":999,\"data_path\":\"/home/trimble/ztbus/compressed/B183_2022-09-21_04-21-57_2022-09-21_17-19-17.csv\"}","level":"info","msg":"starting up","run_id":"lsooPO8","ts":"2023-11-18T16:05:11.681131363Z"}
{"count":46641,"level":"info","msg":"inserting records","run_id":"lsooPO8","ts":"2023-11-18T16:05:11.745933176Z"}
{"body":"{\"bus_id\":\"B183\",\"ts\":\"2022-09-21T04:21:57Z\",\"power\":0,\"altitude\":0,\"route_name\":\"-\",\"passenger_count\":0,\"vehicle_speed\":0,\"traction_force\":0}","headers":"{\"Accept\":[\"application/json\"],\"Authorization\":[\"--redacted--\"],\"Content-Type\":[\"application/json\"]}","host":"localhost:9200","level":"info","method":"POST","msg":"sending request","path":"/ztbus003/_doc","query":"{}","request_id":"bHgknfn","run_id":"lsooPO8","scheme":"https","ts":"2023-11-18T16:05:11.746091851Z"}
{"body":"{\"_index\":\"ztbus003\",\"_id\":\"dkIt44sBo-3F19CozTnQ\",\"_version\":1,\"result\":\"created\",\"_shards\":{\"total\":2,\"successful\":1,\"failed\":0},\"_seq_no\":51294,\"_primary_term\":1}","elapsed":16670921,"headers":"{\"Content-Length\":[\"164\"],\"Content-Type\":[\"application/json\"],\"Location\":[\"/ztbus003/_doc/dkIt44sBo-3F19CozTnQ\"],\"X-Elastic-Product\":[\"Elasticsearch\"]}","level":"info","msg":"received response","path":"/ztbus003/_doc","request_id":"bHgknfn","run_id":"lsooPO8","status":201,"ts":"2023-11-18T16:05:11.762755532Z"}
... ... ... 
{"elapsed":171.118553876,"level":"info","msg":"insertion finished","run_id":"lsooPO8","ts":"2023-11-18T16:08:02.864524793Z"}
```

Oof, that's like, almost 3 minutes.
Yeah, Bulk API ftw?

B-but check out those beautiful continuous-duty logs!
Have a look at the [code](https://github.com/clarktrimble/ztbus/blob/main/cmd/load-ztbus/main.go) for all the gories.

### Aggregate

Now that we've got some data indexed, we'll take a look at an aggregation.
Let's say, ... avgerage speed over some interval for each bus.
Even this can be mildly daunting when confronted by the copious aggregation [documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html) over at Elastic.
A cool trick is to use a Kibana dashboard to cook up something that's close and go from there on the programmatic side.

#### - Kibana Aside -

First of all, I'm a big fan.
I've troubleshot many an issue via Kibana's Discovory mode, which provides a filterable view of records.
We're going to look at Dashboard mode as a means of generating a starter aggregation query.

You might want to skip over this section if you're already cool with the mysteries.


We'll begin by generating an enrollment token and running Kibana in another container:

```bash
docker exec -it es01 /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana

docker run --name kib01 --net elastic -p 5601:5601 docker.elastic.co/kibana/kibana:8.11.1
```

Point a browser to Kibana, plug in the enrollment token, and login as "elastic" user from above.

First create a "Data view":
[instructions](https://www.elastic.co/guide/en/kibana/current/data-views.html), or in bread crumb form:

**3bar (top-left button) > Discover > "Data View Menu" (just below 3bar) > Create a data view**

- Name: something memorable
- Index pattern: something that matches just the index(es) of interest
- Timestamp field: ts
- Save

Now for the aggregation!:

**3bar > Dashboard > Create a Dashboard > Add Panel > Aggregation based > Data table**

Choose the Data view from above and try out an aggregation, for example:
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

Voila!  Kibana has got us off to a good start in the mildly arcane world of ES aggregation queries :)

On with the show.
Golang, golang, golang ..

## Programmatic Aggregation

### Templating

For programmatic aggregations, we'll need a way to generate aggregation queries on the fly for variable date-ranges, etc.
Hmmm, perhaps a template will do the job?:

```yml
---
aggs:
  outer:
    date_histogram:
      field: "ts"
      fixed_interval: "{% raw %}{{ .interval }}{% endraw %}"
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
            gte: "{% raw %}{{ .bgn }}{% endraw %}"
            lte: "{% raw %}{{ .end }}{% endraw %}"
size: 0
```

The query will:
 - filter on a date range
 - bucket the data by interval
 - bucket the interval buckets by bus_id
 - and finally get the average speed for a bus in an interval

Note the presence of the keyword ".keyword".
Recall mention of "configurables below the surface" above?
When the data was thrown at an unconfigured index, ES decided to create the "bus_id" field as `text`, which is not available for term aggregation.
Luckily for us, it also created an nearly identical sub-field "keyword" that _is_ up for term aggregation.

The template is in yaml rather than json to improve readability for complex queries.
This three-layer aggregation is more compact, etc. than its two-layer forbear above.
The wizardry is courtesy [github.com/ghodss/yaml](https://github.com/ghodss/yaml) which leverages the concept that json can be viewed as a subset of yaml.

To pull this off, first the template is rendered and then converted to json:

```go
// template.go
func (tmpl *Template) RenderJson(name string, data any) (out []byte, err error) {

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

Have a closer look at the structure of the query above.
"outer", "middle", and "inner" are values rather than field names.
This plays hell with Marshal/Unmarshal of Golang's json package.

### Road Test

There's a [dump-query](https://github.com/clarktrimble/ztbus/blob/main/cmd/dump-query/main.go) utility in ztbus that dumps the rendered query as well as the response body from ES.

```bash
$ bin/dump-query | jq > out.json
```

[Output](/pdf/dump-query.json) can be voluminous, but very handy when ironing out kinks.


### Result Wrangling

At this point we've got the programmatic aggregation query under control, now to handle the resulting data.
In the home stretch!

Here's a snippet of what we get back:

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

Ouch! Generally, and we have the pesky value-as-json-key thing again.

[github.com/tidwall/gjson](https://github.com/tidwall/gjson) to the rescue:

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

In the `for` loops, we see gjson making short work of pulling the data we're after from the query result and presenting us with a tidy `ztbus.AvgSpeeds` object.

Of course, this code is irretrievably coupled to the `"avgspeed"` aggregation query template but my hope is to isolate this to the `AvgSpeed` method in a service-layer package dedicated to the ZTBus dataset.

### Putting it all Together

```bash
$ ZTB_SVC_DRYRUN=false bin/aggregate 2> agg.log
2022-09-21T08:00:00Z    B183    0.990000
2022-09-21T08:00:00Z    B208    4.930000
2022-09-21T08:05:00Z    B183    1.730000
2022-09-21T08:05:00Z    B208    4.826667
2022-09-21T08:10:00Z    B183    4.440000
2022-09-21T08:10:00Z    B208    3.023333
...
```

Where [aggregate](https://github.com/clarktrimble/ztbus/blob/main/cmd/aggregate/main.go) is printing the results for us in tsv.

## The End (Really)

We've had a whirlwind introduction to ES and its aggregations to set the stage for:

 - templated queries
 - non-marshalled extraction of data from results

Both of which offer a workable, pragmatic even, approach to exposing the considerable power of programmatic aggregation available with Golang.

### Thanks

Whew, another rather code-heavy post.

Thanks for reading!

