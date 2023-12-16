---
title: ConfigState
description: Blurring the line between configuration and program state with discovery.
date: 2023-12-14
tags:
  - golang
  - discovery
  - consul
layout: layouts/post.njk
---

{% image "./discovery.jpg", "Discovering a shipwreck" %}

If you think about it long enough, you can blur the line between configuration and program state; kind of like pointing your index fingers together just a few inches in front of your eyes.
Go ahead, give it a try ... can you see the "sausage"?
Lol, anyway, let's take the example of discoverables.

Suppose we have a PhotoBook service and one of it's many features is to resize photos.
This sort of thing can be compute-intensive and we have broken resize out into its own separately scalable service.
In the beginning PhotoBook can simply be configured with the available resizers.
Later, as things really scale up :), automating this now-significant chore starts to look worthwhile.
If the resize workers register somewhere, PhotoBook can discover them; configstate!

In this post we'll take a look at a Golang [project](https://github.com/clarktrimble/configstate) that tracks available services as found in a Consul key-value store. It also shows off a nice approach to running additional workers in an http api service.

## Consul

Hashicorp's Consul is a solid product that offers service discovery, mesh, and quite a lot more.
I imagine it's usually found deployed in support of their container orchestration system, Nomad.
In the name of simplicity and perhaps portability, we'll make do with Consul's excellent key-value store, which backs it's galaxy of other features.

Starting up a stand-alone Consul and following its logs:

```bash
~/proj/configstate$ docker run -d --name consul01 -p 8500:8500 hashicorp/consul:1.17
~/proj/configstate$ docker logs -f consul01
```

That was easy; thanks Hashi!

### KV Store

Trying out the KV Store:

```bash
~/proj/configstate$ curl -s localhost:8500/v1/kv/bargle -XPUT -d'{"ima":"pc"}' | jq
true
~/proj/configstate$ curl -s localhost:8500/v1/kv/bargle | jq
```
```json
[
  {
    "Key": "bargle",
    "Value": "eyJpbWEiOiJwYyJ9",
    "ModifyIndex": 42526
    ...
  }
]
```

```bash
~/proj/configstate$ curl -s localhost:8500/v1/kv/bargle?raw | jq
{
  "ima": "pc"
}
```

Simples! we can `PUT` a value, `GET` it in its full glory with a base64 encoded value, or just the raw value. 

### Long Polling

_A-and_ we can __long-poll__!:

``` bash
~/proj/configstate$ curl -v "localhost:8500/v1/kv/bargle?index=42526&wait=9s" | jq
...
< X-Consul-Index: 42532
< X-Consul-Knownleader: true
< X-Consul-Query-Backend: blocking-query
...
100   185  100   185    0     0     40      0  0:00:04  0:00:04 --:--:--    40
```

```json
[
  {
    "Key": "bargle",
    "Value": "eyJpbWEiOiJwYyB0b28hIn0=",
    "ModifyIndex": 42532
    ...
  }
]
```

What's going on here:
 - add `index=42526` to the request, telling Consul we want to wait for the next version
 - add `wait=9s` to the request, telling Consul we want to wait _up to_ 9 seconds
 - sneak off to another terminal and `PUT` an updated `bargle` value to Consul (about 4 seconds)
 - Consul responds when the new value is available, with a new index (version) as well
 - Consul will respond with the same old value and index after `wait` if nothing changes

Long-polling can really hit the spot and I think it does here:
 - simple http, easily demo'd with curl
 - values are not expected to change frequently
 - changes are available with minimal delay
 - built in back-stop in case a change "event" is somehow missed
 - a subscriber can load values on startup via the same method as it will poll 

So yeah, a nice way to keep up with dynamic configuration.
I'm having trouble thinking of an example of this that I cannot frame as discovery, but there's nothing here bound to it.

### More Features

Not surprisingly, Consul takes things discovery quite a bit further than the KV Store and you can read up on [Catalog](https://developer.hashicorp.com/consul/api-docs/catalog) and [Services](https://developer.hashicorp.com/consul/api-docs/agent/service) in their excellent api docs.

Another interesting option could be to store services individually and think of them as in a services folder.
This might look like:

```bash
~/proj/configstate$ curl -s localhost:8500/v1/kv/svc/24 -XPUT -d@service24.json
~/proj/configstate$ curl -s localhost:8500/v1/kv/svc?recurse | jq
[
  {
    "Key": "svc/05",
    "ModifyIndex": 11969
    ...
```

### Simple Blob

In our case just a simple blob of structured data under a single key will do.
So we'll go with that!

Putting services into the store:

```bash
~/proj/configstate$ curl -s localhost:8500/v1/kv/services-test -XPUT -d@test/data/services.json | jq
true
```


## Golang

Cool, so we've got a long-pollable key-value store from which we can get a list of services.
How might consuming this look from within a Golang api service?
The [configstate](https://github.com/clarktrimble/configstate) project demonstrates with a minimal api service and an additional worker polling for updates.

Jumping in with an overview of main.go:

```diff-go
func main() {
  // load config and setup logger
  // ...

  // init graceful and create router

  ctx = graceful.Initialize(ctx, &wg, lgr)

  rtr := chi.New()
  rtr.Set("GET", "/config", delish.ObjHandler("config", cfg, lgr))

  // start discovery and register handler

+ client := cfg.ConsulClient.NewWithTrippers(lgr)
+ csl := cfg.Consul.New(client)
+ dsc := &discover.Discover{Poller: csl, Logger: lgr}

+ dsc.Start(ctx, &wg)
+ dsc.Register(rtr)

  // start server and wait for shutdown

  server := cfg.Server.NewWithLog(ctx, rtr, lgr)
  server.Start(ctx, &wg)
  graceful.Wait(ctx)
}
```

After the usual api setup:
 - consul is created with injection of an http client
 - discovery is created with injection of a poller, in this case consul

Once discovery is setup, it starts and registers with router.

Looking at just the discover worker's logs for another high-level perspective, omitting all but the msg field for clarity:

```bash
~/proj/configstate$ bin/discover | jq '. | select(has("worker_id")) | .msg'
"worker starting"
"sending request"
"received response"
"updating services"
"sending request"
< some time passes ... >
< process killed >
"worker shutting down"
"worker stopped"
```

The discover worker starts, gets data, updates its services list, sends another request and .. hangs?

Aha!, this is the long-poll.
It's got the current data and waiting for Consul to respond or timeout.

Then the process is killed externally and it shuts down.
Left to run, we'd see:
 - requests/responses at around the configured poll interval cadence
 - an early response and "updating services" when the data changes
 - perhaps an occasional error from the network or something, followed by continued plodding

To round things out, let's ask the api for services known to it:

```bash
~/proj/configstate$ curl -s localhost:8081/services | jq
```
```json
{
  "services": [
    {
      "uri": "http://pool04.boxworld.org/api/v2",
      "capabilities": [
        {
          "name": "resize",
          "capacity": 23
        }
      ]
    },
    ...
  ]
}
```

And soon after updating the key-value store, those changes will be reflected here.

### Discover

Discover implements a worker that polls for services and provides access to other, hypothetical at this point, subsystems that need the information.

Let's see how it works:

```go
// discover.go , most logging ommitted for reader sanity
func (dsc *Discover) work(ctx context.Context, wg *sync.WaitGroup) {
  dsc.hash = fnv.New64a()
  wg.Add(1)
  defer wg.Done()

  for {
    data, err := dsc.Poller.Poll(ctx)
    if errors.Is(err, context.Canceled) {
      break
    }
    if err != nil {
      continue
    }
    if dsc.unchanged(data) {
      continue
    }

    services, err := entity.DecodeServices(data)
    if err != nil {
      dsc.Logger.Error(ctx, "failed to watch", err)
      continue
    }

    dsc.mu.Lock()
    dsc.services = services
    dsc.mu.Unlock()
  }
}
```

It loops for-ever:
 - getting data from the poller
 - quitting if the context is cancelled
 - logging errors
 - ignoring unchanged data
 - stashing fresh data in a private field

The beating heart of discover!

Of note, is the absence here of any concerns over timing.
The loop will run as fast as it can, relying completely on the poller for good behavior.
This is a design decision, which may stand the test of time :)

Finally, discover provides access:

```go
// discover.go
func (dsc *Discover) Services() entity.Services {

  dsc.mu.RLock()
  defer dsc.mu.RUnlock()

  return dsc.services.Copy()
}
```

With the locking and copying, we see simplicity and reliability prioritized over performance.
Which one can reasonably hope is appropriate for the use case. 

### Poller

A Consul object satisfies the poller interface with:

```go
// consul.go
func (csl *Consul) Poll(ctx context.Context) (data []byte, err error) {

  delay := csl.Limiter.Reserve().Delay()
  csl.LimitDelay += delay
  time.Sleep(delay)

  var newIdx uint64
  data, newIdx, err = csl.GetKv(ctx, csl.Key, csl.Idx)
  if err != nil {
    return
  }

  if newIdx < csl.Idx {
    newIdx = 0
  }
  csl.Idx = newIdx

  return
}
```

Ahh, now we're getting into matters of timing.
The first line of defense is hidden away in `GetKv` which will send requests with `wait` as seen in the curl example above.

Should that fail, say due to a fritzy service re-registering every few nanoseconds, the rate limiter will slow things down.
This makes me happy :)
Any delay that the limiter does introduce is accumulated in `LimitDelay` for some visibility, although I'm not making use of it currently.

Hashi has a nice [page](https://developer.hashicorp.com/consul/api-docs/features/blocking) discussing the use of Consul's blocking endpoints.

### Graceful

At this point I've sketched, hopefully with some verisimilitude, how discover, along with it's consular poller, maintains a dynamic configstate representing discovered services.
Now I'd like to call your attention to the graceful shutdown of a service with multiple workers.

Let's start by having another look at the logs of a trial run, but this time without filtering any messages:

```bash
~/proj/configstate$ bin/discover | jq .msg
"starting up"
"worker starting"
"starting http service"
"listening"
"sending request"
"received response"
"updating services"
"sending request"
< process killed >
"shutting down"
"shutting down http service"
"http service stopped"
"worker shutting down"
"worker stopped"
"stopped"
```

After the process is killed, we see:
 - a general "shutting down" message
 - http service and discover worker shutdown and stop
 - a general "stopped" message before the process exit

In all honesty, we could very likely get away with just pulling the rug from under these two.
This is a good practice though and often-times crucial to the prevention of insidious errors in the larger system.

These conveniences are brought to us by `graceful` from the delish module:

```go
// graceful.go
func Initialize(ctx context.Context, wg *sync.WaitGroup, lgr Logger) context.Context {
  ctx, cancel := context.WithCancel(ctx)

  graceful = &Graceful{
    Cancel:    cancel,
    WaitGroup: wg,
    Logger:    lgr,
  }

  return ctx
}
```

When the graceful singleton is initialized it stashes:
 - `CancelFunc` for the context
 - reference to the wait group
 - a logger

Once all the workers are running, graceful can wait for an interrupt:

```go
// graceful.go
func Wait(ctx context.Context) {

  sigChan := make(chan os.Signal, 1)
  signal.Notify(sigChan, stop...)
  <-sigChan

  graceful.Logger.Info(ctx, "shutting down ..")

  graceful.Cancel()
  graceful.WaitGroup.Wait()

  graceful.Logger.Info(ctx, "stopped")
}
```

First, it does the usual signal channel blocking to wait.

Once the fix is in, it:
 - cancels the context, signaling any participating workers to shutdown
 - waits for all of them to report that they have finished by decrementing the wait group

Sweet!

If you look back up at discover's work loop, you'll see:

```go
  ...
  if errors.Is(err, context.Canceled) {
    break
  }
  ...
```

Triggering it to shutdown.

Somewhat similarly, in the delish http service, you'll see:

```go
  ...
  <-ctx.Done()
  svr.Logger.Info(ctx, "shutting down http service ..")
  err := httpServer.Shutdown(ctx)
  ...
```

The difference here is because the discover worker spends its time waiting for `Poll` to return while the http service can simply wait for the `Done` channel to close.

## The End

To sum up, the configstate project demonstrates:
 - long-polling for simple dynamic updates
 - service discovery decoupled from a particular source
 - graceful shutdown with multiple workers

I hope it's been informative and/or thought provoking.
Get in touch with me via email if you'd like :)

Thanks for reading!










