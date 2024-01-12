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
In addition to sporting the operable features shown off in [Continuous Duty CLI](https://clarktrimble.online/blog/continuous-duty-cli/) (most notably logging), the service:
 - registers with discovery
 - responds to a `/monitor` endpoint
 - responds to a `/metrics` endpoint

For discovery we'll return to HashiCorp's lovely Consul service, but this time go beyond it's humble key-value store and register as a service with its agent api.

For stats collection, we'll opt for the ubiquitous Prometheus, discovering our service via Consul.

And from there, who can resist a Grafana for painless, svelt visualization?
Not me, lol.

## An Observable Service

Just to mix things up, we'll start by going over the [code](https://github.com/clarktrimble/stator) before bringing everything up for a demo.

__But first, what and why observable?__

A reasonable take on an observable serivce could be one that registers its presence and reports it's health by way of logging and responding to related endpoints.

As an infrasctucture scales up, such observability becomes more and more a factor in being able to efficiently operate.
And everyone, senior managment included, loves a good visualization.

### Registration

At first blush, this looks like a simple api request on startup, but what about _when_ the discovery service loses it's mind and forgets about us?
(Generally any event used to establish state wants to be backed by a synchronizing true-up.)
A blunt, simple solution is to re-register periodically, and we'll go with that.

Starting the registration service:

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
An intrepid troubleshooter uniformly provided with the likes can quickly filter and pivot thier way to a picture of "what happened?".

Of course, depending on a particular logger is a faux pas extraordinare in some circles and you'll be glad to know `Logger` shows up only as an interface in the roster package.

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

When `ticker` fires, re-register, _or_ when the context's `cancelFunc` is called (from a goroutine elsewhere), unregister and exit, but not before decrementing the waitgroup.

A tidy worker indeed, but what about the actual register/unregister?
Let's take a quick peek at `unregister` as it's the more interesting case:

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
Squeaked by here, as `WithoutCancel` is new for go1.21; thanks Golang maintainers!

Back to those turtles:

```go
// roster.go
type Registrar interface {
  Register(ctx context.Context, svc entity.Service) (err error)
  Unregister(ctx context.Context, svc entity.Service) (err error)
}
```

Ahh, `Registrar` shows up as an interface in `roster`.
It's reasonable to suppose this lets us change our mind and use something else for discovery instead.
It could happen.

B-but the immediate gold-plated payoff is in testibility.
Roster's worker is hairy enough to properly unit test and it helps heaps that we can stop at simply checking that our mock was called as expected.

Registering and Deregistering with Consul:

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

Form up some data and make a request!

Relying on the `Client` interface provides for testing with nary a Consul service running, woot.

I wonder if `SendObject` and its marshalling help might be a bit much for some grizzled gophers.
Nice to have that tucked away somewhere though?
And we get request/response logging in the implementation injected in main.

We'll finish up registration with a snippet from main:

```go
  // main.go
  rtr := minroute.New(ctx, lgr)
  rtr.HandleFunc("GET /monitor", delish.ObjHandler("status", "ok", lgr))

  client := cfg.Client.NewWithTrippers(lgr)
  csl := cfg.Consul.New(client)
  rstr := cfg.Roster.New(cfg.Server.Port, csl, lgr)

  rstr.Start(ctx, &wg)
```

Setup a monitor endpoint and start the triple-injected roster service.

Check out the previous post on [Encapsulated Environmental Configuration](https://clarktrimble.online/blog/encapsulated-env-cfg/) to see more regarding `cfg`.

### Metrics Endpoint

When a request for stats comes in, we'll collect, format and respond.

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

While this could be construed as overkill just to expose a few runtime stats, a service layer of this sort helps to keep things from getting jammed together.

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

Collect, format, write; a sensible handler, downshifting from the world-wide-web with aplomb.
`runCollectors` and `format` call the interface methods as you might expect.

There's a convinience method bringing things back to earth for the observaiblity case:

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

I would like to highlight the `HandleFunc` method in the router interface.
It's meant to anticipate the new mux coming with go1.22.
Thanks to Eli Bendersky for a nice [post](https://eli.thegreenplace.net/2023/better-http-server-routing-in-go-122/) regarding this exciting develpment!

### Runtime Stats

Total memory allocated, cpu time, goroutine count, and so on can provide assurance that a Golang executable is behaving well.
Traditionally these have been available via `runtime.ReadMemStats`, but go1.16 reduced overhead significantly with [runtime/metrics](https://pkg.go.dev/runtime/metrics).

Trying it out:

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

We're specifying which stats to collect as a variable in the `runtime` package.
Totally unconfigurable, sigh, but quite workable for many cases.

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

Setup a sample slice per stdlib `metrics`, read, and convert to our intermediate representation. 

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
Storing the values as strings here would have worked as well, but hanging on to the numbers a bit longer "feels" right.

Another thing emerging from the above sprawl, is the structuring of related points in two layers.
`PointsAt` holds common infomations and `Points` the gritty details.

Having belabored the runtime collector so thouroughly, I think we'll skip over the Prometheus formatter (think `fmt.Sprintf` and `strings.Builder`).

And at long last, a snippet from main putting a few collectors to work:

```go
// main.go
  svc := stator.ExposeRuntime(appId, runId, rtr, lgr)
  svc.AddCollector(cfg.DiskUsage.New())
  svc.AddCollector(wave.New())
```

Exposing the runtime metrics is a one-liner and I've snuck in a couple additional collectors to spice things up.

Disk usage is interesting not only in the sense that running out of space is a longtime fan-favorite failure mode, but it also provides an example of points which differ only by labels.

Wave is a toy collector that emits series of sine waves, giving us something fun to visualize.
I did have a good time writing it, and a predictable signal could be useful for integration testing.

### Metric or Log ?

A quick word about what's not being collected.
Requests over time per endpoint and the time taken to handle them for example.
They're both potentially of some interest, yeah?

I'd say so and the good news is they can be aggregated from simple-to-emit request/response logs.
In this way, relatively perishible logging serves multiple purposes before disposal and contributes to a sleek implementation.

Often times there's murk in the metric vs log question with no single correct answer.
With no need to get it "right", we can adjust for other factors in the infrastructure or whatever feels like a good balance.

### Endpoint Trial Run

Running `stator` and hitting the metrics endpoint with curl:

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

Looks plausible : )

Now we turn our attention to Consul, Prometheus, Grafana and give `stator` a proper workout!

## Compose

With four services in the mix, we'll bring in docker-compose to help manage the development environment.

The compose file:

```yaml
// docker-compose.yaml
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
    ## show below
  godev:
    ## show below

networks:
  labneh:
```

Not only do we get a slathering of much needed organization, but compose lets us refer to services by name from other containers in the same network.
For example, from the `godev` container:

```bash
/project $ curl -s consul:8500/v1/catalog/services
{"consul":[]}
```

We can use "consul" as the hostname given to curl, sweet!

## Discovery

Starting up a stand-alone Consul with compose:

```bash
$ docker-compose up -d consul
$ docker logs -f stator_consul_1 # as needed
```

That was easy, thanks Hashi!

## Go Develpment

We'll deploy on Alpine for a small container image, so we'll compile in an Alpine container (libc musl).

From the compose file:

```yaml
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
```

The dockerfile:

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

We mount the project and the go package module folders in the container.
The idea here is to run `git` from the host system, sparing us the headache of dealing with auth from within the container.
The `devo` user is created with our host user id so that file ownership is consistent.


Building the `godev` container image:

```bash
$ docker-compose build --build-arg userid=$(id -u) godev
```

This'n could be built elsewhere and pulled in from a container registery.

## Stator

From the compose file:

```yaml
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
```

The dockerfile copies the compiled binary into an Alpine.

Note that we're sneaking in the config via `env_file`. Here's a peek:

```bash
STTR_SERVER_PORT=8087
STTR_LOGGER_MAXLEN=999
STTR_CLIENT_BASEURI=http://consul:8500
STTR_ROSTER_SERVICE_ID=123654
STTR_ROSTER_SERVICE_NAME=stator
STTR_ROSTER_SERVICE_TAGS=dev,metrics
```

Cool, let's build `stator` and its container image, a-and start it up:

```bash
$ docker-compose run godev make          # run tests and compile binary in Alpine
$ docker-compose build --no-cache stator # build image for deployment
$ docker-compose up -d stator            # start the service!
$ docker logs -f stator_stator_1         # checkout the logs as needed
```

Browsing over to `http://your-docker-host-here:8500` :

{% image "./consul-stator.png", "Consul web ui showing registration of stator service" %}

`stator` has registered and Consul is keeping an eye on it for us through the `/monitor` endpoint.

Just like that, our service and Consul are nestled into their own docker network!

## Collection

We'll need to configure Prometheus to look for services in Consul:

```yaml
# prometheus.yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "api_services"
    consul_sd_configs:
      - server: 'consul:8500'
        tags:
          - "metrics"
```

The "api_services" job says to scrape Consul services with the "metrics" tag.

And turn the compose crank again:

```bash
$ docker-compose build prom
$ docker-compose up -d prom
$ docker logs -f stator_prom_1 # as needed
```

Browsing over to `http://your-docker-host-here:9090`, with any luck:

{% image "./prom-stator.png", "Prometheus web ui showing stator service as a target" %}

Sweet scalable action!
Prometheus is polling `stator` having discovered it via Consul.
Neither know nothin' about our service (in terms of static config I should say).
Assuming a production Consul cluster and a couple of beefy Prometheans, many stators can be observed.

Both Consul and Prometheus are, not surprisingly, ready and willing to send alerts when configured with the appropriate conditions.

## Vizualization

I've always liked you Grafana ...

```bash
$ docker-compose build graf
$ docker-compose up -d graf
$ docker logs -f stator_graf_1 # as needed
```

Browse on over to `http://your-docker-host-here:3000`,

login (admin/admin) > add Prometheus data source > Explore > choose a metric > click Run Query and:

{% image "./grafana-stator.png", "Grafana web ui showing stator service memory stats" %}

Looks like `stator` is settling into a modest pile of RAM after start-up, thank goodness!

And for a little fun let's see how the sine wave collector turned out:

{% image "./grafana-wave.png", "Grafana web ui showing stator service wave stats" %}

Yellow is the sum of three random sine waves.

Green is the first four of the Fourier series converging on a square wave.
Isn't it amazing that just four sine waves get this close to a square?
I think I've been in love with applied math since seeing a demo of this in third-grade, awww.

## The End

Wow, mega-post!

To sum up, `stator` achieves observability by:
 - registering with discovery
 - responding to a monitor endpoint
 - responding to a metrics endpoint
 - punctilious logging

Such observability contributes to efficienctly operating at scale.

Per usual, I hope it's been informative and/or thought provoking.
Feel free to get in touch with me via email if you'd like : )

Thank you for reading!










