---
title: Stator
description: An observable Golang service.
date: 2024-01-09
tags:
  - golang
  - discovery
  - metrics
layout: layouts/post.njk
---

{% image "./stator.png", "A large electric generator" %}

In this post, we'll take a look at an implementation of an observable Golang api service.
In addition to sporting the operable features shown off in [Continuous Duty](https://clarktrimble.online/blog/continuous-duty-cli/), the service:
 - registers with discovery
 - responds to a `/monitor` endpoint
 - is discovered by stats collection
 - responds to a `/metrics` endpoint

For discovery we'll return to HashiCorp's lovely Consul service, but this time go beyond it's humble key-value store and dive into service discovery proper.

For metrics we'll turn to the ubiquitous Prometheus.

And from there, who could resist a little Grafana for painless, svelt visualization.

## An Observable Service

Just to mix things up, we'll take a look at the [code] before bringing up the related services via Docker.

__But first, what and why observable?__

A reasonable take on an observable serivce could be one that registers its presence and reports it's health.
As an infrasctucture scales up, such observability becomes more and more a factor in being able to efficiently operate.
And, of course, everyone, senior managment included, loves a good visualization.

### Registration

At first blush, this looks like a simple api "call" on startup, but what about _when_ the discovery service loses it's mind and forgets about us, or everything?
(Generally any event used to establish state will need to be backed by a synchronizing true-up.)

A blunt, simple solution is to re-register periodically, and we'll take this approach here.
One can imagine scenarios where a service needs to update metadata about itself with discovery, or maybe discovery being discomfited by a superflous re-register, but these are left as exercises for the reader today.

Ok, so we'll register on startup and re-register every so often:

```go
func (roster *Roster) Start(ctx context.Context, wg *sync.WaitGroup) {
  ctx = roster.Logger.WithFields(ctx, "worker_id", hondo.Rand(7))
  roster.Logger.Info(ctx, "worker starting", "name", "roster")

  roster.register(ctx)
  go roster.work(ctx, wg)
}
```

Register and start a worker in a goroutine, presumably to re-register, yawn, but check out that logging!

The `Logger.WithFields` call adds a unique(ish) `worker_id` field to all messages logged with the returned `ctx`.
An intrepid troubleshooter will have such for filtering as needed.
Of course, depending on a particular logger is a faux pas extraordinare in some circles and you'll be glad to know `Logger` shows up as an interface in the roster package.

Now to re-register:

```go
func (roster *Roster) work(ctx context.Context, wg *sync.WaitGroup) {
  wg.Add(1)
  defer wg.Done()

  ticker := time.NewTicker(roster.Interval)

  for {
    select {
    case <-ticker.C:
      roster.register(ctx)

    case <-ctx.Done():
      roster.unregister(ctx)
      roster.Logger.Info(ctx, "worker stopped")
      return
    }
  }
}
```

Golang channels at thier classic best! :)

When the `ticker` fires, register (again), or when `ctx`'s `cancelFunc` is called (from a goroutine elsewhere), unregister and exit, but not before decrementing the waitgroup.

A tidy little worker indeed, but what about the actual register/unregister?

Let's take a quick peek at `unregister` as it's a little more interesting:
```go
func (roster *Roster) unregister(ctx context.Context) {
  ctx = context.WithoutCancel(ctx)

  err := roster.Registrar.Unregister(ctx, roster.Service)
  if err != nil {
    roster.Logger.Error(ctx, "failed to unregister", err)
  }
}
```

Turtles all the way down!!?

Kinda, but first let me draw your attention to the `WithoutCancel` wrinkle.
`ctx` is here so that we benifit from the logging fields hidden within, but it has already been cancelled, which can cause a problem in the http client that eventually sends the unregister request.
Scraped by here as `WithoutCancel` is new as of go1.21; thanks Golang maintainers!

Back to those turtles:

```go
type Registrar interface {
  Register(ctx context.Context, svc entity.Service) (err error)
  Unregister(ctx context.Context, svc entity.Service) (err error)
}
```

Ahh, `Registrar` shows up as an interface in `roster`.

It's reasonable to suppose this lets us change our mind and use etcd or something for discovery instead, could happen.
And it's nice to separate reg/unreg and worker logics.

B-but the immediate gold-plated payoff is in testibility.
Roster's worker is hairy enough to properly unit test and it helps heaps that we can stop at simply checking that our mock was called as expected.

With an interface it can be as easy as:

```go
//go:generate moq -out mock_test.go . Registrar Logger

rc := registrar.RegisterCalls

Expect(rc()).To(HaveLen(1))
Expect(rc()[0].Ctx).To(Equal(ctx))
Expect(rc()[0].Svc).To(Equal(svc))
```

([moq]() and [gomega]() ftw!)

Finally, actually (un)registering with Consul:

```go
type Client interface {
  SendObject(ctx context.Context, method, path string, snd, rcv any) (err error)
}

func (csl *Consul) Register(ctx context.Context, svc entity.Service) (err error) {
  reg := register{
    ID:      svc.NameId(),
    Name:    svc.Name,
    // more fields ...
    },
  }
  err = csl.Client.SendObject(ctx, "PUT", registerPath, reg, nil)
  return
}

func (csl *Consul) Unregister(ctx context.Context, svc entity.Service) (err error) {
  err = csl.Client.SendObject(ctx, "PUT", fmt.Sprintf(unregisterPath, svc.NameId()), nil, nil)
  return
}
```

With the above benifits of resting upon a `Client` interface.
While `SendObject` and its marshal/unmarshal help might be a bit much for grizzled gophers, I think I can make a solid case for the request/response logging seen in the implementation injected in main below.

I very much appreciate a simple-to-use api such as offerred by Consul (not at all looking at etcd v3 here;).

Wrapping up registration with a snippet from main:

```go
client := cfg.Client.NewWithTrippers(lgr)
csl := cfg.Consul.New(client)
rstr := cfg.Roster.New(cfg.Server.Port, csl, lgr)

rstr.Start(ctx, &wg)
```

Triple injected goodness :)
There's a previous [post](https://clarktrimble.online/blog/encapsulated-env-cfg/) regarding the approach to configuration in play.

## Metrics

When a request comes in, we'll collect, format and respond with them.

The baseline metrics for an observable Golang service can be from the runtime package.
Traditionally these have been available via 'runtime.ReadMemStats', but go1.16 reduced overhead significantly with [runtime/metrics].
A few stats like ... will help us to feel good about our services behavior and health in the wild.

Lets look at the metrics service layer:

```go
type Collector interface {
  Collect(time.Time) (pa entity.PointsAt, err error)
}
type Formatter interface {
  Format(pa entity.PointsAt) (data []byte)
}
type Router interface {
  HandleFunc(pattern string, handler http.HandlerFunc)
}
type Logger interface {
  Error(ctx context.Context, msg string, err error, kv ...any)
}

type Svc struct {
  Collectors []Collector
  Formatter  Formatter
  Logger     Logger
}
```

While this could be construed as overkill just to expose a few runtime stats, service layers of this sort keep things from getting jammed together and of course the usual interfacy benifits accrue.

Of particular note is `Collectors` which let us plug-in anything that can round up a few points.
Certainly overkill for runtime stats, but a portion of my motivation is to work through the general case for when one of the "kitchen-sink" collectors like [node-exporter](https://github.com/prometheus/node_exporter) is not a good fit.

Actually do something (handle a request):
```go
func (svc *Svc) GetStats(writer http.ResponseWriter, request *http.Request) {
  ctx := request.Context()

  stats := svc.runCollectors(ctx)
  data := svc.format(stats)

  _, err := writer.Write(data)
  if err != nil {
    svc.Logger.Error(ctx, "failed to write stats to response", err)
  }
}
```

Collect, format, write; a sensible handler, downshifting from the world wide web with aplomb.

`runCollectors` and `format` call the interface methods as you might expect.

Lets take a look at a convinience method bringing things back to earth for our use case:

```go
func ExposeRuntime(appId, runId string, rtr Router, lgr Logger) (svc *Svc) {

  svc = &Svc{
    Collectors: []Collector{
      runtime.Runtime{AppId: appId, RunId: runId},
    },
    Formatter: prometheus.Prometheus{},
    Logger:    lgr,
  }

  rtr.HandleFunc("GET /metrics", svc.GetStats)
  return
}
```

Slot in the runtime collector, specify prometheus format and set a route for our handler, yay!

I would like to point out the `HandleFunc` method in the router interface.
It's meant to anticipate the new mux coming with go1.22.
Thanks to Eli for a nice [post](https://eli.thegreenplace.net/2023/better-http-server-routing-in-go-122/) regarding this exciting develpment!




## The End

I hope the iterative nature today hasn't been too much of an exposé, lol.

And, as always, I hope it's been informative and/or thought provoking.
Get in touch with me via email if you'd like :)

Thanks for reading!










