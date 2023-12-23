---
title: Generic Golang
description: Cleaning up a ugly bit of code with a dash of generics.
date: 2023-12-19
tags:
  - golang
  - generics
  - clickhouse
layout: layouts/post.njk
---

{% image "./generics.png", "Generic Brand Groceries" %}

I've been on the lookout for a good opportunity to wield the "new" generics in Golang since they made their surprise appearance in Go 1.18. (/sarc)
I have noticed them getting put to good use here and there in libraries and for a functional Golang style, [github.com/samber/lo](https://github.com/samber/lo) really puts the generics hammer down, nice.

Anyway, my recent explorations with ClickHouse turned up just such an opportunity.
In the post on [column-orientation](https://clarktrimble.online/blog/funhouse/), I wound up with two less-than-satisfactory approaches to using the low-level `ch-go` library:
 1. reflect over a struct to divine column names and go forth with complexity
 2. stuff ClickHouse logic into a package really about messages, simple but ugly

In this post I'll share how a small dollop of generics triggered a series of improvements, transforming the duckling into, if not a swan, maybe a Barnacle goose. (Really, they have them in Finland!)

## The Ugly

As we're looking at improvement over time, a couple of references into the project's history might be helpful:
 - [old and simple/ugly](https://github.com/clarktrimble/funhouse/tree/de8f5d339a36fe55d48bc66d4aa540e3969c6f7f) circa November 7, 2023
 - [new and improved](https://github.com/clarktrimble/funhouse/tree/2f4b9cfce47c857647944cb9e97a7dd50caef699) circa Dec 21, 2023

Now, lets get our hands dirty! :)

To set the stage, have a look at the old `OnResult` callback used when chunking results out of ClickHouse:

```go
// msgtable.go
err = client.Do(ctx, ch.Query{
  Body:   fmt.Sprintf(qSpec, tableName),
  Result: results,
  OnResult: func(ctx context.Context, block proto.Block) error {

    mcs.Length += block.Rows
    for _, col := range results {
      switch col.Name {
      case "ts":
        mcs.Timestamps = append(mcs.Timestamps, fl.Dt64Values(col.Data)...)
      // ... more cols
      }
      col.Data.Reset()
    }

    return mcs.CheckLen()
  },
})
```

Each element of `results` holds a block of query results for a particular column.
We identify each by name and copy the block into the output struct, `mcs`.
Ok, not too terrible.

Taking a closer look at the `Dt64Values` helper:

```go
// funlite.go
func Dt64Values(cr proto.ColResult) (vals []time.Time) {
  ca, ok := cr.(*proto.ColDateTime64)
  if !ok {
    return
  }

  vals = make([]time.Time, ca.Rows())
  for i := 0; i < ca.Rows(); i++ {
    vals[i] = ca.Row(i)
  }
  return
}
```

It pulls data from the result column and returns it in an appropriately typed slice.

_Oh_, we'll need one of these for each type, not so good : (

_And_ we build a slice here only to append to the slice we really want after returning : /

Eventually, wandering the voluminous and sparsely documented halls of `ch-go`, I had all the helpers looking more or less the same as above.
They call `Rows()` for length and `Row()` for data, varying in the type asserted and the type returned.

Feels like something a generic could help with .. ?

## Generic, Round One

Generics let us define an interface like:

```go
type Rower[T any] interface {
  Rows() int
  Row(i int) T
}
```

Which says, roughly, a Rower, of some type, is that which implements:
 - `Rows()` returning an int
 - `Row(i int)` returning the same type specified for Rower

With `Rower` we can write a helper like:

```go
func Append[T any](slice *[]T, rr Rower[T]) {

  for i := 0; i < rr.Rows(); i++ {
    *slice = append(*slice, rr.Row(i))
  }
}
```

Which says, roughly, for a slice of some type, and a rower of the same type, append row values to slice.

Only one helper needed and we append directly to the output slice!

Here's how processing the "ts" column looks with a generic `Append` helper:

```go
  case "ts":
    fl.Append(&mcs.Timestamps, col.Data.(*proto.ColDateTime64))
```

Better :)

Shout out to [Eli Bendersky](https://eli.thegreenplace.net/2021/generic-functions-on-slices-with-go-type-parameters/) for a nice article on generic slices.  Thanks Eli!

### TL;DR

This section is the punchline for making the world a better place through the judicious application of generics.
From here, I follow my nose to a few more or less related improvements to the project.

## Generic Round Two

Getting this far, traipsing around the `ch-go` docs again, I stumbled upon:

```go
type ColumnOf[T any] interface {
  Column
  Append(v T)
  AppendArr(v []T)
  Row(i int) T
}
```

Where `Column` winds up specifying `Rows() int`.

Aha!, no need for my quaintly named `Rower` interface, we can use `proto.ColumnOf` from the ClickHouse lib.

Lingering over the `ColumnOf` interface just a moment, this, not surprisingly, makes sense.
`ColumnOf` includes methods where we need to know the type.
While `Column` gives us an interface free of such concerns.

Which gets me thinking about the type assertions when copying between cols and data struct seen above.
Not pretty, and the implausibly attentive among you will have noticed that "Round One" now crashes on an unexpected column result type : (

## Concrete Column Types

Hmmm, to avoid the assertion we need a reference to the concrete column type.

They've been defined with:

```go
  dataCols  = map[string]proto.Column{
    "ts": (&proto.ColDateTime64{}).WithLocation(time.UTC).WithPrecision(proto.PrecisionNano),
    // ... more cols
  }
```

The idea here was to use the same definition to generate `proto.Input` and `proto.Results` which _is_ nice, but the concrete types disappear within `map[string]proto.Column`.


Rearranging a little:

```go
type Cols struct {
  Ts *proto.ColDateTime64
  // ... more cols
}

type MsgTable struct {
  Name string
  Cols Cols
  Data *entity.MsgCols
}
```

We can still herd the columns into an `Input` or `Results` struct for use with `ch.Client.Do` and wherever we've got an instance of MsgTable we can refer to the concrete type.

Here's how appending from the "ts" column looks with a concrete type handy:

```go
  case "ts":
    flt.Append(&mt.Data.Timestamps, mt.Cols.Ts)
```

Even better :)

Of course there's nothing safe about concurrent use of a MsgTable instance, but we are quite a lot better off with respect to the issue now they're in the struct rather than a package var õ_0.

## Prising Msg and ClickHouse Logics

With the above in place, separating these two becomes irresistible.

First we pull the ClickHouse concerns out of the `msgtable` package:

```go
// funlite.go
func (fh *Fh) GetResults(ctx context.Context, tbr Tabler) (err error) {

  results, err := results(tbr)
  if err != nil {
    return
  }

  err = fh.Client.Do(ctx, ch.Query{
    Body:   fmt.Sprintf("select * from %s", tbr.TableName()),
    Result: results,
    OnResult: func(ctx context.Context, block proto.Block) error {

      return tbr.AppendFrom(block.Rows, results)
    },
  })
  return
}
```

`GetResults` takes a `Tabler` interface and taps out to its `AppendFrom` in the callback.

Here is `MsgTable`'s implementation:

```go
// msgtable.go
func (mt *MsgTable) AppendFrom(count int, results proto.Results) (err error) {
  mt.Data.Length += count

  for _, col := range results {
    switch col.Name {
    case "ts":
      flt.Append(&mt.Data.Timestamps, mt.Cols.Ts)
    // ... more cols
    }
    col.Data.Reset()
  }
  return mt.Data.CheckLen()
}
```

With something quite similar on the insert side, where "chunking" code makes more of an appearance.

Cool, it's starting to feel like everything has and is in its place! :)

## Leaving Room for Improvement

But first, summing up the improvements:
 - replace per-type helpers with one generic helper
 - eliminate intermediate slice through which all results were copied
 - creep towards intended use of lib with `proto.ColumnOf`
 - concrete column types
 - reusable ClickHouse chunking logic

Funhouse is shaping up, but I cannot say I'm completely, like, in love.

### Tag Driven

My first effort at this was overly complex and I'm glad to have shelved it, but .. it's still sitting there on the shelf.
The idea is to drive the code connecting a struct to `ch-go` via tags, à la `json`, etc.

```go
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

The appeal is a much [smaller](https://github.com/clarktrimble/funhouse/blob/2f4b9cfce47c857647944cb9e97a7dd50caef699/examples/reflecting/msgtable/msgtable.go) supporting `MsgTable` that can focus on table'y things.

### Queries

So far the queries supported are rather bland:

```go
  err = fh.Client.Do(ctx, ch.Query{
    Body:   fmt.Sprintf("select * from %s", tbr.TableName()),
    Result: results,
    // ...
```

This is partly due to the lack of an actual use case, yeah just screwing around so far :), and "query builders" tend towards the unsatisfying to implement?

However, _there is_ a larger issue lurking here.
If the columns in `results` don't match the columns specified in the query's select phrase, `ch.Client.Do` returns an error.
I'm thinking of two different fixes:
 1. make the results helper "select" aware
 2. query only one column at a time

The second might work out?
I don't know, but it does highlight an aspect of ClickHouse's column-orientation which is that you don't pay for unused columns when querying.

## The End

I hope the iterative nature today hasn't been too much of an exposé, lol.

And, as always, I hope it's been informative and/or thought provoking.
Get in touch with me via email if you'd like :)

Thanks for reading!










