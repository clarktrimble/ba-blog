---
title: FunHouse
description: ClickHouse and Low-Level Golang
date: 2023-11-08
tags:
  - golang
  - database
layout: layouts/post.njk
---

In this post we'll take a look at ClickHouse and explore it's columnar nature via Golang.

{% image "./clickhouse.png", "Abstract Columns" %}

## ClickHouse

ClickHouse is billed as a scalable backend for large amounts of data coming at you fast.
Like maybe trillions of records arriving in the 100's of millions per second!

Quantities of this sort strongly imply logs and stats, although the sample datasets in the docs go well beyond.
I'm thinking of where ClickHouse would be an attractive alternative to more established technologies such as Elasticsearch for logs and Prometheus for stats.

### Kool-Aid

Kool-Aid down the hatch?
Almost certainly -- look for the "warts" post some months from now : )

For the time being, ClickHouse "feels" good:

 - plenty of GitHub stars
 - interesting, informative documentation
 - obsession/passion with performance
 - practical and technology driven culture

A quote from their "Why so fast?" page:

> Last but not least, the ClickHouse team always monitors the Internet on people claiming that they came up with the best implementation, algorithm, or data structure to do something and tries it out. Those claims mostly appear to be false, but from time to time you’ll indeed find a gem.

### Column Oriented

Many aspects contributing to the performance talked up by ClickHouse are beyond the scope of this post, and "column-oriented" is in the tag-line and therefore likely to be a heavy hitter.
Columns of a table are stored apart from each other, which allows for better compression and so forth.

Where column-oriented started to click for me was in a gem of an article about [sparse primary indexes](https://clickhouse.com/docs/en/optimize/sparse-primary-indexes).
In describing how the primary and secondary indexes behave, one is necessarily exposed to the granular, or blocky, nature of how data is written and read.
Individual rows are not directly accessible.
The closest you can get is to process the block in which a row is to be found.
Typically these blocks, or granules, are as large as 8192 records, almost 3 orders of magnitude.

Of course, you can write a query that returns a single row, but ClickHouse will grab at least one block, or quite possibly all of them if your indexing doesn't support it, and discard the rest of the data.
However, when you want data in bulk, a system built around getting and putting blocks of it can out-perform one that is not.

OK, column-oriented bulk, this is good for ?.., well -- analysis.
Say you've got weblogs of the form: timestamp, domain, path, response code, and elapsed time.
An aggregation can be used to transform the logged data into stats like requests per unit time, or paths taking longer than they did yesterday to respond.
And those or other stats can be rolled up for higher-level views.
Weblogs are the proverbial data firehose, so handling them with efficiency and performance in mind will be advantageous.

So, yeah, starting to get a feel for column-oriented, but let's dig a little more.

## Golang Client

When trying to get to know a new back-end, there's nothing like starting an instance and programmatically writing and reading a little data.

Turning to the Golang hammer, we have a choice of client libraries!:

### clickhouse-go

[clickhouse-go](https://github.com/ClickHouse/clickhouse-go) is a high-level client with lots of features, including support a native interface and the standard `database/sql` interface.
This is an attractive option with plenty of examples for both.

### ch-go

[ch-go](https://github.com/ClickHouse/ch-go/tree/main) is a low-level client with few features and not many examples.
It is said to be designed for performance and seems to focus on the ClickHouse protocol.

Looks like a good choice for exploring and learning .. coin flip .. ch-go it is! (for this post)

## FunHouse

I cleverly name my exploratory project FunHouse which can be found on [GitHub](https://github.com/clarktrimble/funhouse/tree/main) for reference.

### First Steps

#### Step Zero: get ClickHouse running

```bash
docker run -d \
  --name ch-dev \
  --ulimit nofile=262144:262144 \
  --network=host \
  --cap-add=SYS_NICE --cap-add=NET_ADMIN --cap-add=IPC_LOCK \
  clickhouse/clickhouse-server
```

I appreciate being able to get a standalone dev server running without a lot of fuss.

Thanks ClickHouse!

#### Step One: put some stuff in

Cut and paste from the [insert example](https://github.com/ClickHouse/ch-go/blob/main/examples/insert/main.go) in the ch-go repo and .. it works!

I think.  Let's take a peek with the CLI client to be sure:

```bash
$ docker exec -it ch-dev clickhouse-client

tartu.boxworld.com :) select * from test_table_insert

SELECT *
FROM test_table_insert

Query id: ad629ba8-7ad2-4595-b357-62f263021d99

┌────────────────────────────ts─┬─severity_text─┬─severity_number─┬─body────────────┬─name────┬─arr───────────┐
│ 2023-11-07 22:20:55.571613007 │ INFO          │               3 │ body from colzz │ name-0  │ ['sna','foo'] │
│ 2023-11-07 22:20:55.571613032 │ INFO          │               3 │ body from colzz │ name-1  │ ['sna','foo'] │
│ 2023-11-07 22:20:55.571613049 │ INFO          │               3 │ body from colzz │ name-2  │ ['sna','foo'] │
...
```

Yes!

#### Step Two: get stuff out (programmatically)

Um, surprisingly, I'm not finding an example for this one. Challenge accepted!

### Wrong Turn

The ch-go example code cited above can be fairly described as in need of a little factoring.
Gods know, I love a good factoring.
So after getting something working on the read side, I began to factor.
What started with reusing the same `proto.Column`'s for both `input` and `results` became a complete separation of any details for a particular struct/table from a reusable corpus of `funhouse` code including reflection over "col" tags.

I had fun.
I stretched; creating a "col" tag with reflection was a new one for me.
It works!
Maybe it's even a good start on something worthy?

In the end, I preferred the considerably more accessible, if clunkier, code from earlier in the project.
We'll stick with this simpler, kinder version of the code for the remainder of the post. 

## Column-Oriented Golang

Let's take a look at a row-oriented struct vs a column-oriented struct representing messages.

Here's row-oriented:

```go
// entity/msg.go
type Severity struct {
        Txt string
        Num uint8
}
type Msg struct {
        Timestamp time.Time
        Severity  Severity
        Name      string
        Body      string
        Tags      []string
}
```

One instance corresponds to one row.

And the equivalent column-oriented one:

```go
// entity/msg.go
type MsgCols struct {
        Length       int
        Timestamps   []time.Time `col:"ts"`
        SeverityTxts []string    `col:"severity_text"`
        SeverityNums []uint8     `col:"severity_number"`
        Names        []string    `col:"name"`
        Bodies       []string    `col:"body"`
        Tagses       [][]string  `col:"arr"`
}
```

Nothing mind-blowing; the slices surely are column-oriented.

If need be, we can easily transform between the two.


#### TL;DR

As we'll see below, the column-oriented struct comes in handy with `ch-go`.
If you're in a hurry, it's a good summary of "column-oriented" writ small in Golang.

### Insert / Put

Diving deeper into the Golang, the `PutColumns` function inserts messages into the table a chunk at a time:

```go
// examples/generable/msg/msg.go
func PutColumns(ctx context.Context, client *ch.Client, chunkSize int, mcs *entity.MsgCols) (err error) {
  err = mcs.CheckLen()
  if err != nil {
    return // slices must have the expected len
  }

  idx := 0
  input := fl.Input(colNames, dataCols)

  err = client.Do(ctx, ch.Query{
    Body:  input.Into(tableName),
    Input: input,
    OnInput: func(ctx context.Context) error {

      input.Reset() // zero cols for next chunk
      if idx > mcs.Length {
        return io.EOF // all finished
      }
      end := min(idx+chunkSize, mcs.Length)

      dataCols["ts"].(*proto.ColDateTime64).AppendArr(mcs.Timestamps[idx:end])
      dataCols["severity_text"].(*proto.ColEnum).AppendArr(mcs.SeverityTxts[idx:end])
      // ... additional columns ommitted for clarity ...

      idx += chunkSize
      return nil // send a chunk
    },
  })
  return
}
```

The main thing going on here is that `client.Do` calls the `OnInput` handler until `EOF` or some other error is returned.
On each pass, a block of data is sent to the server, which it can gobble with column-oriented efficiency.

From the above, it's not obvious that `input` and `dataCols` share the same underlying `proto.Columns`.
A look at the `Input` method should throw some light on the matter:

```go
// funlite/funlite.go
func Input(names []string, byName map[string]proto.Column) (input proto.Input) {
  input = proto.Input{}
  for _, name := range names {
    input = append(input, proto.InputColumn{
      Name: name,
      Data: byName[name],
    })
  }
  return
}
```

Along with a very similar `Results` function, this lets us avoid repeating ourselves when it comes to delineating the columns we might want to insert or read to/from a table.

### Read / Get

The `GetColumns` function reads messages from the table a block at a time:

```go
// examples/generable/msg/msg.go
func GetColumns(ctx context.Context, client *ch.Client, qSpec string) (mcs *entity.MsgCols, err error) {
  mcs = &entity.MsgCols{}
  results := fl.Results(colNames, dataCols)

  err = client.Do(ctx, ch.Query{
    Body:   fmt.Sprintf(qSpec, tableName),
    Result: results,
    OnResult: func(ctx context.Context, block proto.Block) error {

      mcs.Length += block.Rows
      for _, col := range results {
        switch col.Name {
        case "ts":
          mcs.Timestamps = append(mcs.Timestamps, fl.Dt64Values(col.Data)...)
        case "severity_text":
          mcs.SeverityTxts = append(mcs.SeverityTxts, fl.EnumValues(col.Data)...)
        // ... additional columns omitted for clarity ...
        }
        col.Data.Reset()
      }

      return mcs.CheckLen()
    },
  })
  return
}
```

The tricky part for me was to (mostly) ignore `block` in the handler and read from the enclosed `results`.
Again, the data is handled block-wise enabling the potential for column-oriented efficiency.

The per-column append lines use type-specific helpers.
Let's have closer look at one of them:

```go
// funlite/funlite.go
func Dt64Values(cr proto.ColResult) (vals []time.Time) {
  ca, ok := cr.(*proto.ColDateTime64)
  if !ok {
    return
  }

  vals = make([]time.Time, cr.Rows())
  for i := 0; i < ca.Rows(); i++ {
    vals[i] = ca.Row(i)
  }
  return
}
```

One of these is be needed for each type of `proto.Column` used in a table.

## The End

Whew; plenty of code there!
I hope it helps to convey a sense about column-orientation.

The rubber meets the road in the `OnInput` and `OnResults` callbacks.
Both move blocks of data via the column-oriented `MsgCols` struct.

### Leaving Room for Improvement

#### Performance

I've made an effort to keep things trim, but the code presented is unoptimized.
It would be interesting to profile and tweak.

I'd also like to push quite a bit more data.

#### Generate / Generic

Would be nice to demonstrate generating the table/struct specific code along with providing tests.
First though, I want to scratch my head a little about bringing generics to bear.
I suspect the commonality of calling `AppendArr` or `Rows/Row` after the type assertions makes this promising.

#### Indexing

Would be informative to create indexes and do something more than select all.
Especially pushing things to the point where additional tables are needed.

### Thanks

Hey, we made it to the end!
Thank you for reading : )

