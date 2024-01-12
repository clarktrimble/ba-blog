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

For discovery we'll return to HashiCorp's lovely Consul service, but this time go beyond it's humble key-value store and register as a service with its agent api.

For stats collection, we'll opt for the ubiquitous Prometheus, discovering our service via Consul.

And from there, who could resist a little Grafana for painless, svelt visualization.

## An Observable Service

Just to mix things up, we'll take a look at the [code] before bringing up all the services for a demo.

__But first, what and why observable?__

A reasonable take on an observable serivce could be one that registers its presence and reports it's health and status by way of logging and responding to endpoints appropriately.

As an infrasctucture scales up, such observability becomes more and more a factor in being able to efficiently operate.
And, of course, everyone, senior managment included, loves a good visualization.

### Registration

At first blush, this looks like a simple api request on startup, but what about _when_ the discovery service loses it's mind and forgets about us, or everything?
(Generally any event used to establish state wants to be backed by a synchronizing true-up.)

A blunt, simple solution is to re-register periodically, and we'll take this approach here.
One can imagine scenarios where a service needs to update metadata about itself with discovery, or maybe discovery is discomfited by superflous re-registers, but these are left as exercises for the reader.

Ok, so we'll register on startup and re-register every so often:

```go
// roster.go
func (roster *Roster) Start(ctx context.Context, wg *sync.WaitGroup) {
  ctx = roster.Logger.WithFields(ctx, "worker_id", hondo.Rand(7))
  roster.Logger.Info(ctx, "worker starting", "name", "roster")

  roster.register(ctx)
  go roster.work(ctx, wg)
}
```

Register and start a worker in a goroutine, presumably to re-register, yawn, but check out that logging!

The `Logger.WithFields` call adds a unique `worker_id` field to all messages logged with the returned `ctx`.
An intrepid troubleshooter uniformly provided with such can quickly filter and pivot thier way to a picture of "what happened?".

Of course, depending on a particular logger is a faux pas extraordinare in some circles and you'll be glad to know `Logger` shows up as an interface in the roster package.

Now to re-register:

```go
// roster.go
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

Golang channels at thier classic best!

When `ticker` fires, re-register, _or_ when `ctx`'s `cancelFunc` is called (from a goroutine elsewhere), unregister and exit, but not before decrementing the waitgroup.

A tidy little worker indeed, but what about the actual register/unregister?
Let's take a quick peek at `unregister` as it's a little more interesting:

```go
// roster.go
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
`ctx` is passed in so that we benifit from the logging fields hidden within, but it has already been cancelled, which can cause a problem in the http client that eventually sends the unregister request.
Squeaked by here, as `WithoutCancel` is new as of go1.21; thanks Golang maintainers!

Back to those turtles:

```go
// roster.go
type Registrar interface {
  Register(ctx context.Context, svc entity.Service) (err error)
  Unregister(ctx context.Context, svc entity.Service) (err error)
}
```

Ahh, `Registrar` shows up as an interface in `roster`.

It's reasonable to suppose this lets us change our mind and use etcd or something for discovery instead.
It could happen, and it's nice to separate worker and register logics.

B-but the immediate gold-plated payoff is in testibility.
Roster's worker is hairy enough to properly unit test and it helps heaps that we can stop at simply checking that our mock was called as expected.

With an interface it can be as easy as:

```go
// roster_test.go
//go:generate moq -out mock_test.go . Registrar Logger

rc := registrar.RegisterCalls

Expect(rc()).To(HaveLen(1))
Expect(rc()[0].Ctx).To(Equal(ctx))
Expect(rc()[0].Svc).To(Equal(svc))
```

([moq](https://github.com/matryer/moq) and [gomega](https://onsi.github.io/gomega/#top) ftw!)

Registering and Deregistering with Consul in particular:

```go
// consul.go
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

Resting on a `Client` interface here allows for testing with nary a Consul.

I wonder if `SendObject` and its marshal/unmarshal help might be a bit much for grizzled gophers.
Nice to have that tucked away somewhere though?
And I think I can make a solid case for the request/response logging seen in the implementation injected in main below.

I very much appreciate a simple-to-use api such as offerred by Consul (not at all looking at you etcd v3 ;).

Wrapping up registration with a snippet from main:

```go
// main.go
client := cfg.Client.NewWithTrippers(lgr)
csl := cfg.Consul.New(client)
rstr := cfg.Roster.New(cfg.Server.Port, csl, lgr)

rstr.Start(ctx, &wg)
```

Triple injected goodness :)

Check out [Encapsulated Environmental Configuration](https://clarktrimble.online/blog/encapsulated-env-cfg/) if you'd like to know more more about `cfg`.

## Metrics

When a request for stats comes in, we'll collect, format and respond.

The baseline metrics for an observable Golang service can be had from the runtime package.
Traditionally these have been available via `runtime.ReadMemStats`, but go1.16 reduced overhead significantly with [runtime/metrics](https://pkg.go.dev/runtime/metrics).
A few stats like ... will help us to feel good about our services behavior and health in the wild.
// Todo: rewrite ^^^

Diving into the code with a look at the service struct:

```go
// stator.go
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

While this could be construed as overkill just to expose a few runtime stats, a service layer of this sort keep things from getting jammed together and of course the usual interfacy benifits accrue.

Of some note is `Collectors` which let us plug-in anything that can round up a few stats.
Certainly overkill for runtime stats, but a portion of my motivation is to work through the general case for when one of the "kitchen-sink" collectors like [node-exporter](https://github.com/prometheus/node_exporter) is not a good fit.

Alrighty, lets handle a request:
```go
// stator.go
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
// stator.go
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
Thanks to Eli Bendersky for a nice [post](https://eli.thegreenplace.net/2023/better-http-server-routing-in-go-122/) regarding this exciting develpment!

Now we'll turn out attention to the runtime metrics themselves:

```go
// runtime.go
var (
  collectible = []metric{
    {
      long: "/cpu/classes/total:cpu-seconds",
      name: "cpu_total",
      unit: "seconds",
      desc: "Total available CPU time",
    },
    // more stats ...
}
```

We're specifying which stats to collect as a variable in our runtime package.
Simple stuff, but it's fun to think about a use case where we had a need to handle dynamically ala [ConfigState](https://clarktrimble.online/blog/configstate/).

Go forth and collect!:

```go
// runtime.go
func (rt Runtime) Collect(ts time.Time) (pa entity.PointsAt, err error) {
  samples := make([]metrics.Sample, len(collectible))
  for i := range collectible {
    samples[i].Name = collectible[i].long
  }
  metrics.Read(samples)

  points, err := toPoints(samples)
  if err != nil {
    return
  }

  pa = entity.PointsAt{
    Name:  name,
    Stamp: ts,
    Labels: entity.Labels{
      {Key: "app_id", Val: rt.AppId},
      {Key: "run_id", Val: rt.RunId},
    },
    Points: points,
  }
  return
}
```

Build a sample slice per `runtime/metrics` in the stdlib, read, and convert them to our intermediate "entity".

To round things out, the points helper:

```go
// runtime.go
func toPoints(samples []metrics.Sample) (points []entity.Point, err error) {
  points = make([]entity.Point, len(collectible))
  for i, sample := range samples {

    var value entity.Value
    switch sample.Value.Kind() {
    case metrics.KindUint64:
      value = entity.Uint{Data: sample.Value.Uint64()}
    case metrics.KindFloat64:
      value = entity.Float{Data: sample.Value.Float64()}
    default:
      err = errors.Errorf("unknown go runtime stat type for: %s", sample.Name)
      return
    }

    points[i] = entity.Point{
      Name:  collectible[i].name,
      Desc:  collectible[i].desc,
      Unit:  collectible[i].unit,
      Type:  "gauge",
      Value: value,
    }
  }
  return
}
```

A lot of column-inches in the last two blocks!
But hopefully easy enough to read. :)

Of some small interest, is the `Value` interface in `Point`.
It embeds `fmt.Stringer` and supports ints and floats for now.
Stringifying the values here would have worked as well, but hanging on to the numbers a little longer "feels" right.

Another thing emerging from the above sprawl, is the structuring of related points in two layers.
`PointsAt` holds common infomations and `Points` the gritty details.
Seems to be working out well!

Having belabored the runtime collector so thouroughly, I think we'll skip over the prometheus formatter and it's many `Sprintf`s and string builders.

And at long last a snippet from main putting all this to work:

```go
// main.go
  svc := stator.ExposeRuntime(appId, runId, rtr, lgr)
  svc.AddCollector(cfg.DiskUsage.New())
  svc.AddCollector(wave.New())
```

Exposing the runtime metrics is a one-liner and I've snuck in a couple additional collectors to spice things up.

Disk usage is interesting not only in the sense that running out is still a fan-fav failure mode, but it also provides a nice example of points which differ only by labels.

Wave is kind of a toy collector that emits series of sine waves, giving us something fun(?) to look at.
I did have a good time writing it :), a-and a simple sine wave actually could be useful for integration testing.

#### not collected at all

Before we wrap up collection, a quick word about what's not being collected.

For an api service, requests per endpoint and time spent handling those requests are _rather_ important metrics.
I've not mentioned them til now, because that information is in the logs and in my somewhat humble opinion stats which can be aggregated from the logs are best collected in such a manner.  Not trying to be orthodox or anything; I'm espousing the idea as it seems so much easier to fire and forget a log message rather than tracking all that within an otherwise focused service.

#### the metrics

Before getting Prometheus and the other supporting services running, lets start `stator` and check the metrics endpoint with curl:

```bash
$ make                # test and build stator repo
$ bin/stator -h       # show configurables
$ source etc/dev.sh   # set env with dev config
$ bin/stator -c       # show loaded config
$ bin/stator          # run the service!
```

```bash
$ curl localhost:8087/metrics # from another term

# HELP gort_cpu_total_seconds Total available CPU time
# TYPE gort_cpu_total_seconds gauge
gort_cpu_total_seconds{app_id="stator",run_id="YNwO5Ul",process_id="175005"} 0.00 1704906955008

# HELP gort_mem_total_bytes All memory mapped into the current process
# TYPE gort_mem_total_bytes gauge
gort_mem_total_bytes{app_id="stator",run_id="YNwO5Ul",process_id="175005"} 7969808 1704906955008

# HELP gort_goroutines_count Count of live goroutines
# TYPE gort_goroutines_count gauge
gort_goroutines_count{app_id="stator",run_id="YNwO5Ul",process_id="175005"} 10 1704906955008
### more go runtime stats ...

# HELP du_used_percent Percentage of space on the filesystem in use
# TYPE du_used_percent gauge
du_used_percent{path="/"} 16.66 1704906955008
du_used_percent{path="/boot/efi"} 1.14 1704906955008
### more disk usage stats ...

# HELP wave_sine Sine wave(s)
# TYPE wave_sine gauge
wave_sine{name="three_random"} -0.17 1704906955008
wave_sine{name="square"} 0.39 1704906955008
```

Looking plausible :)

Now we turn our attention to Consul, Prometheus, and Grafana so we can give it a proper workout!

## Compose

With four services, including `stator`, in the mix, we'll bring in docker-compose to help manage the environment.
Here's the compose file:

```yaml
services:
  consul:
    image: hashicorp/consul:1.17.1
    ports:
    - 8500:8500
    networks:
    - labneh
  prom:
    build:
      context: .
      dockerfile: etc/prom.Dockerfile
    ports:
    - 9090:9090
    networks:
    - labneh
  graf:
    image: grafana/grafana-enterprise:10.2.3
    ports:
    - 3000:3000
    networks:
    - labneh
  stator:
    build:
      context: .
      dockerfile: etc/stator.Dockerfile
    env_file:
      etc/stator.env
    ports:
    - 8087:8087
    networks:
    - labneh
  godev:
    build:
      context: .
      dockerfile: etc/godev.Dockerfile
    networks:
    - labneh
    volumes:
    - .:/project
    - ~/go/pkg/mod:/go/pkg/mod
    working_dir: /project

networks:
  labneh:
```

Not only do we get a slathering of much needed organization, but compose lets us refer to services by name from other containers in the same network.
For example, from the `godev` container:

```bash
/project $ curl -s consul:8500/v1/catalog/services
{"consul":[]}
```

We were able to use "consul" as the hostname given to curl, nice!

A few noteables from the compose file:
 - `stator`'s config is specified via `env_file`
 - the current dir from host (repo) is mounted as "/project" in `godev`
 - the go pkg dir from host is mounted in `godev`, effectively caching


## Discovery

Our observable service needs a discovery service with which to register :)

Starting up a stand-alone Consul with compose:

```bash
$ docker-compose up -d consul
$ docker logs -f stator_consul_1 # as needed
```

That was easy, thanks Hashi!

## Stator

This one's a little more involved.
We're going to deploy on Alpine for a small image, so we'll build Golang in Alpine as well (libc musl you know).

Here's the `godev` dockerfile:

```dockerfile
FROM golang:1.21.6-alpine3.19
LABEL description="Golang development"

RUN apk add --no-cache \
    curl \
    ### a few more pkgs ...
ARG userid
RUN addgroup -S -g ${userid} devo
RUN adduser -S -u ${userid} -g devo devo
USER devo

RUN go install github.com/matryer/moq@v0.3.3
RUN curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | \
    sh -s -- -b $(go env GOPATH)/bin v1.55.2

WORKDIR /project
CMD sh
```

Bog standard, but for the user hand-wring, explanation just below ..

Now to build the container image:

```bash
$ docker-compose build --build-arg userid=$(id -u) godev
```

Aha! We're passing in our user id from the host system as a "build arg".
This is to get file ownership lined up between host and container in mounted volumes.

`godev` only needs to be built once, or when changes are made to its dockerfile.

Cool, let's build `stator` and its container image, a-and start it up:

```bash
$ docker-compose run godev make
$ docker-compose build --no-cache stator
$ docker-compose up -d stator
$ docker logs -f stator_stator_1 # as needed
```

Just like that, our service and Consul are nestled into their own little docker network!

Browsing over to `http://your-docker-host-here:8500`, behold:

{% image "./consul-stator.png", "Consul web ui showing registration of stator service" %}

`stator` has registerd and Consul is keeping an eye on it for us.

## Collection

We'll need a little config to tell Prometheus to look for stuff in Consul:

```yaml
# prometheus.yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "stator"
    consul_sd_configs:
      - server: 'consul:8500'
        services:
          - "stator"
```

And turn the compose crank:

```bash
$ docker-compose build prom
$ docker-compose up -d prom
$ docker logs -f stator_prom_1 # as needed
```

Browsing over to `http://your-docker-host-here:9090`, with any luck:

{% image "./prom-stator.png", "Prometheus web ui showing stator service as a target" %}

Sweet scalable action!
Prometheus is polling `stator` having discovered it via Consul.
Neither know nothin' about our particular instance (in terms of static config I should say).
Assuming a production Consul cluster and a couple of beefy Prometheans, many stators can be observed.

## Vizualization

I've always liked you Grafana ...

```bash
$ docker-compose build graf
$ docker-compose up -d graf
$ docker logs -f stator_graf_1 # as needed
```

Browse on over to `http://your-docker-host-here:3000`, login with admin/admin, add Prometheus as a data source, Explore, choose "gort_mem_total_bytes" and:

{% image "./grafana-stator.png", "Grafana web ui showing stator service memory stats" %}

Looks like `stator` is settling into a modest pile of RAM after start-up.

And for a little fun let's see how the sine wave collector turned out:

{% image "./grafana-wave.png", "Grafana web ui showing stator service wave stats" %}

The yellow is three random sine waves and green is the first four of the Fourier series converging on a square wave.
Isn't it amazing that just four of them get this close to a square?
I think I've been in love with applied math since seeing a demo of this way back in middle school. :)

## The End

Wow, mega-post!
To sum up, `stator` achieves observability by:
 - registering with discovery
 - punctisomthingy logging
 - responding to a monitor endpoint
 - responding to a metrics endpoint with info on how well it's behaving

As such we could run thousands of them across a global infrastructure and keep tabs on all of 'em.

Per usual, I hope it's been informative and/or thought provoking.
Get in touch with me via email if you'd like :)

Thanks for reading!










