---
title: Encapsulated Environmental Configuration
description: Exporing the upside of envconfig in Golang.
date: 2023-10-19
tags:
  - golang
layout: layouts/post.njk
---

In this post we'll take a look at passing configuration via environmental variables and an approach to encapsulating configuration in Golang.

```

    ┌─────────────────┐                     ┌────────────────────────────────┐
    │ package one     │                     │ package main                   │
    │                 │                     │                                │
    │ Config{         ├────────────────────►│ Config{                        │
    │  Port: int      │                     │  One: *one.Config              │
    │ }               │     ┌──────────────►│  Two: *two.Config              │
    │                 │     │               │  DryRun: bool                  │
    └─────────────────┘     │               │ }                              │
                            │               │                                │
                            │               │ envconfig.Process("pfx", &cfg) │
    ┌─────────────────┐     │               └────────────────────────────────┘
    │ package two     │     │
    │                 │     │
    │ Config{         ├─────┘                Usage:
    │  User: string   │                      KEY                TYPE
    │ }               │                      PFX_DRYRUN         Bool
    │                 │                      PFX_ONE_PORT       Integer
    └─────────────────┘                      PFX_TWO_USER       String

```

## Configuration via Environment

The idea here is to pass config to an application by way of envars rather than command line options or file.

Command line options get out of hand along with the number of configurables.
As a project scales up, the cost of also managing config files scales as well.
Envars are simple key-value pairs just there in, ah, the environment.

Scaled infrastructures built on containerization have good support for environmental config, perhaps even favoring the approach.
The generally sane Twelve-Factor App site has a good [take](https://12factor.net/).
In particular, I'll draw attention to:

> it’s easy to mistakenly check in a config file to the repo

From which I'll extract a small corollary: config needs to be managed separately from an app's codebase.
Except, always the exception(s) of course, maybe some example and/or test config.

## Golang

Okay, okay; we'll take it as given that config via environment is at least workable.
How does this look with our beloved Golang?

```go
val, ok := os.LookupEnv(key)
if !ok {
  fmt.Printf("%s not set\n", key)
} else {
  fmt.Printf("%s=%s\n", key, val)
}

// or if unset and blank are equivelant, simply:
al = os.Getenv(key)
mt.Printf("%s=%s\n", key, val)
```

Easy enough, but we wouldn't want to see that sort of thing in main.
Enter the excellent `envconfig` module!

### envconfig

The [envconfig](https://github.com/kelseyhightower/envconfig) module "decodes" environment variables sharing a prefix into a struct provided by the app.
This is similar to the familiar parsing of json into a struct with `Unmarshal`.

I like the single-responsibility feel of `envconfig` and have had good results with it.

And check out the `go.mod` (no dependencies!):

```go
module github.com/kelseyhightower/envconfig
```

Introductions accomplished, let's have a quick walk through:

```go
/*
  export PFX_PORT=8080
  export PFX_DBUSERNAME=midboi
  export PFX_DRYRUN=true
*/
type Config struct {
  Port       int
  DbUserName string
  DryRun     bool
}

cfg := Config{}
err := envconfig.Process("pfx", &cfg)
```

When all goes well, `cfg` will be left looking like:

```go
main.Config{
  Port:       8080, 
  DbUserName: "midboi", 
  DryRun:     true,
}
```

Sweet!
Notice that `envconfig` is handling type conversion sensibly.

With a little tagging of the struct, `envconfig` can produce helpful usage:

```go
type Config struct {
  Version    string `ignored:"true"`
  Port       int    `desc:"port on which to listen" default:"8083"`
  DbUserName string `desc:"db service acct name" required:"true"`
  DryRun     bool   `json:"dry_run" desc:"dig up metadata, but don't post"`
}

cfg := Config{}
err := envconfig.Usage("pfx", &cfg)
```

```bash
KEY               TYPE             DEFAULT    REQUIRED    DESCRIPTION
PFX_PORT          Integer          8083                   port on which to listen
PFX_DBUSERNAME    String                      true        db service acct name
PFX_DRYRUN        True or False                           dig up metadata, but don't post
```

I often find myself invoking `Usage` to refresh my memory for a particular app and quite frequently copying and pasting from there for a quick list of the variable names.

### Aside on Command Line

During development I keep an env file handy.

```bash
export PB_MATCH='PXL_20230[67]'
export PB_TRUNCATE=99
export PB_DRYRUN="true"
```

Sourcing and overriding as needed:

```bash
. etc/dev-env.sh

PB_DRYRUN=false go run cmd/load/main.go
```

### Aside on Defaults

Full disclosure: I am a recovering default value denier.

This almost certainly comes from operating a system long ago which had many sources for default configuration through multiple templating systems.
It was a real chore to track down where a default was being set and then very possibly more of one to determine the most correct place to make a change.
We did not have a use case for something that flexible, etc., but paid for it on a regular basis.

In light of all that, I find that a `struct` definition is a reasonable place to define defaults, where sensible ones exist.

### Another Quicker Aside on Logging

It can be a quite helpful to know what configuration an app was running when it did something worthy of investigation.
So it's a good idea to log an app's config as part of start-up, appropriately redacted of course.
Representing the configuration with a single structure make this a doddle.

Anyway, on with the show!

## Encapsulation

Consider gentle reader, if you will, a real world example:

```go
type Config struct {
  Version  string         `json:"version" ignored:"true"`
  Truncate int            `json:"truncate" desc:"truncate log fields beyond length"`
  Bolt     *bolt.Config   `json:"bolt"`
  Server   *delish.Config `json:"server"`
}
```

Aha!, a package can define it's configuration requirements and that can be added to main's simply by adding a field.

Let's have a closer look at the server's config defined in `delish`:

```go
// from delish.go
type Config struct {
  Host    string        `json:"host" desc:"hostname or ip for which to bind"`
  Port    int           `json:"port" desc:"port on which to listen" required:"true"`
  Timeout time.Duration `json:"timeout" desc:"characteristic timeout" default:"10s"`
}
```

This is _good_; main or whatever doesn't have to "know" anything about it's dependencies other than that it has them.

As a flourish, we can define another `NewServer` function with `Config` as the receiver for noise reduction in main.

```go
// also from delish.go
func (cfg *Config) NewServer(handler http.Handler, lgr Logger) *Server {
  return NewServer(cfg.Host, cfg.Port, handler, lgr)
}
```

How does `Usage` look with encapsulation in play?:

```bash
KEY                  TYPE        DEFAULT    REQUIRED    DESCRIPTION
PB_TRUNCATE          Integer                            truncate log fields beyond length
PB_BOLT_PATH         String                 true        path to db file (inculsive)
PB_SERVER_HOST       String                             hostname or ip for which to bind
PB_SERVER_PORT       Integer                true        port on which to listen
PB_SERVER_TIMEOUT    Duration    10s                    characteristic timeout
```

Very serviceable :) Notice how `envconfig` adds `SERVER` and a separator.

... if we provide an option for dumping the config as json for a quick sanity check:

```json
{
  "version": "main.24.3735d77",
  "truncate": 99,
  "bolt": {
    "path": "photo.db"
  },
  "server": {
    "host": "",
    "port": 8088,
    "timeout": 30000000000
  }
}
```

Easy on the eyes and eminently loggable.

Now suppose we've been injecting our dependencies as we should, are over the low-level thrill of bolt, and want to try cockroach:

```diff-go
type Config struct {
  Version  string         `json:"version" ignored:"true"`
  Truncate int            `json:"truncate" desc:"truncate log fields beyond length"`
- Bolt     *bolt.Config   `json:"bolt"`
+ Roach    *roach.Config  `json:"roach"`
  Server   *delish.Config `json:"server"`
}
```

Voila! `Config` is doing its part to support something like clean architecture and generally not getting in the way.

### Aside on "Real World" Example

The examples in this section are taken from [pbs](https://github.com/clarktrimble/pbs), a small project in support of a photobook front-end app.
There you'll see `envconfig` lightly wrapped by the [launch](https://github.com/clarktrimble/launch) module.
`launch` provides a few conveniences, such as `-h` and `-c` flags which output usage and config respectively, as seen above.

## Conclusion

 - environment variables are a workable approach to config, perhaps even preferred at scale
 - `envconfig` is a tidy module, capable of carrying the water for Golang
 - mapping config into a top-level struct allows for encapsulation and simple reporting

### Thanks

Hey you made it to the end!
Hopefully Ba Blog will have a feedback concept soon, lol.

Thank you for reading : )

