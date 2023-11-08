---
title: FunHouse
description: ClickHouse and Low-Level Golang
date: 2023-11-08
tags:
  - golang
  - database
layout: layouts/post.njk
---

This is a quick post relating my adventures trying out ClickHouse via their low-level golang client ch-go.

## ClickHouse

ClickHouse is billed as a scalable backend for large amounts of data coming at you fast.
Like maybe trillions of records arriving in the 100's of millions per second!

Quantities of this sort strongly imply logs and stats, although the rather interesting sample datasets in their docs go well beyond.
General data-science aside, my first take is that ClickHouse could be an alternative to Elasticsearch for logs and and Prometheus for stats.

### Kool-Aid
Kool-Aid down the hatch?
Almost certainly; look for the ClickHouse warts post some months from now :)

For the moment, ClickHouse "feels" good:

 - plenty of GitHub stars
 - interesting, informative documentation
 - obsession with performance
 - practical and technology driven culture
 - easy to get a dev instance running

Like everyone else, they're monetezing and the website is littered with prompts to try the Service and integrations.
Hopefully, this will work out.

### Column Oriented

Many aspects contributing the performance claimed by ClickHouse are beyond the scope of this post, but my guess is that "column-orientation" is the killer feature.
Columns of table are stored apart from each other, which allows for better compression, etc; ok.

Where column-oreiented started to click for me was in a gem of an article about sparse primary indexes.
In describing how the primary and secondary indexes work in ClickHouse, one is necessarily exposed to the granular, or blocky, nature of data IO in the bowels.
Individual rows are not directly accessable.
The closest you can get is the block in which a row is to be found.
Aha, block -> columns!!





After becoming disenchanted with the idea of using Dgraph to quickly standup a GraphQL service to support frontend development, I thought Protobuff!

I am interested in API-first and Protobuf likely earns a top-five spot in a list of such things.
And, as a longtime Golang enthusiast, I've cherished a notion or two about Protobuff for years.  Forward!

Spoiler: I'm not convinced gRPC/Protobuff is the best transport for web applications.
Please see far below for rationale and nuances.

Disclaimer:  I'm parking this project for now, so the code you'll see below is rough.
Will update the post if ever cleaned up and published to github.
Additionally, I've hacked away auth and CORS protection for demonstration purposes only.

Credit: I got started by reading: https://medium.com/@aravindhanjay/a-todo-app-using-grpc-web-and-vue-js-4e0c18461a3e .
Vue and Envoy portions of that article are dated at this point.

## Overview

{% image "./overview.png", "Overview" %}

Both the web application and Golang server are informed of the API specification by way of code generated from a Protobuf proto file.

Pretty straightforward, but yeah, why's the Envoy Proxy sitting in the middle of things?

As I understand, broswer-side Javascript is unable to speak full HTTP2 so a translation layer is needed.
Even better, gRPC is built on HTTP2, so from the browser we use grpc-web, which is similar to gRPC.
See https://grpc.io/blog/state-of-grpc-web/ for more.

## Proto

It all begins with an API spec!  In this case, I'm imagining a simple photo gallery:

```protobuf
syntax = "proto3";
package photo;
option go_package = "./photopb";

message getPhotoParams{}
message addPhotoParams {
  string largeName = 2;
  int32 width = 4;
  int32 height = 5;
}
message deletePhotoParams {
  string id = 1;
}
message photoObject {
  string id = 1;
  string largeName = 2;
  int32 width = 4;
  int32 height = 5;
}
message photoResponse {
  repeated photoObject photos = 1;
}
message deleteResponse {
  string message = 1;
}

service photoService {
  rpc addPhoto(addPhotoParams) returns (photoObject) {}
  rpc deletePhoto(deletePhotoParams) returns (deleteResponse) {}
  rpc getPhotos(getPhotoParams) returns (photoResponse) {}
}
```

"photoService" specifies our service, letting us add, delete, and get photos in terms of messages defined above.
The numbers (1-5 in this example) are how gRPC identifies fields on the wire for super compactness.

## Golang Service

We'll use Golang for the backend and even try it out with a quick client.

### Generate Code!

Step zero; download the Protobuf code generator and a couple of plugins:

```bash
mkdir protoc-download
cd protoc-download
curl -LO https://github.com/protocolbuffers/protobuf/releases/download/v24.3/protoc-24.3-linux-x86_64.zip
unzip protoc-24.3-linux-x86_64.zip
cp bin/protoc ~/mymymy/bin/.

go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

Note that protoc is a binary-only download :(

Ok, now generate code!!

```bash
protoc --go_out=. --go-grpc_out=. photo.proto
ls
 photo_grpc.pb.go
 photo.pb.go
```

The "go_out" option uses protoc-get-go to generate photo.pb.go which defines messages specified in the proto.

The "go-grpc_out" options uses proct-gen-go-grpc to generate photo_grpc.pb.go which defines a client and server implementing the service specified in the proto.

Protoc supports different invokation styles and there's more than one way, yay.

### Handler

We can use the generated code to cook up a handler which implements methods from the API specification as needed by whatever particular project (business logics!).

```go
package photopb
import (
  "golang.org/x/net/context"
)

type Server struct {
  Photos []*PhotoObject
  UnimplementedPhotoServiceServer
}

func (srv *Server) AddPhoto(ctx context.Context, newPhoto *AddPhotoParams) (po *PhotoObject, err error) {
  po = &PhotoObject{
    Id:        "asdfasdfasdf",
    LargeName: newPhoto.LargeName,
    Width:     newPhoto.Width,
    Height:    newPhoto.Height,
  }
  srv.Photos = append(srv.Photos, po)
  return
}

func (srv *Server) GetPhotos(ctx context.Context, _ *GetPhotoParams) (*PhotoResponse, error) {
  return &PhotoResponse{Photos: srv.Photos}, nil
}
```

In real life, we might use a repo interface to pop in the DB of our choice.
For today a slice of photos will do for the "database".

### Server

And use the handler in a runnable gRPC server:

```go
package main
import (
  ...
  "xform/photopb"
)

func main() {
  lis, err := net.Listen("tcp", fmt.Sprintf("0.0.0.0:%d", 14586))
  if err != nil {
    panic(err)
  }

  so := grpc.Creds(insecure.NewCredentials())
  grpcServer := grpc.NewServer(so)

  srv := photopb.Server{}
  photopb.RegisterPhotoServiceServer(grpcServer, &srv)

  reflection.Register(grpcServer)

  if err := grpcServer.Serve(lis); err != nil {
    panic(err)
  }
}
```

Apologies for the naming : /

### Client

Hacking up a Golang client is super easy from here; how can we resist?:


```go
package main

import (
  ...
  "xform/photopb"
  "xform/takeout"
)

func main() {

  // scratch up some photo metadata and connect gRPC client
  photos, err := takeout.Find("/home/trimble/takeout01")
  if err != nil {
    panic(err)
  }

  conn, err := grpc.Dial("localhost:14586", grpc.WithTransportCredentials(insecure.NewCredentials()))
  if err != nil {
    panic(err)
  }
  defer conn.Close()
  client := photopb.NewPhotoServiceClient(conn)

  // add some photos
  for _, photo := range photos {
    pbPhoto := &photopb.AddPhotoParams{
      LargeName: fmt.Sprintf("http://tartu/photo/resized/%s-4.png", photo.Name),
    }
    _, err = client.AddPhoto(context.Background(), pbPhoto)
    if err != nil {
      panic(err)
    }
  }

  // turn around and get photos
  pbPhotos, err := client.GetPhotos(context.Background(), &photopb.GetPhotoParams{})
  if err != nil {
    panic(err)
  }
  for _, pbPhoto := range pbPhotos.Photos {
    fmt.Printf(">>> %#v\n", pbPhoto.LargeName)
  }
}
```

### Test Drive

Run the server:

```bash
export GRPC_GO_LOG_VERBOSITY_LEVEL=99
export GRPC_GO_LOG_SEVERITY_LEVEL=info
go run examples/server/main.go
 2023/09/22 16:09:16 INFO: [core] [Server #1] Server created
 2023/09/22 16:09:16 INFO: [core] [Server #1 ListenSocket #2] ListenSocket created
```

Yeah, built in logging.
I wonder why they don't give us an interface instead?

Anyway, it's alive!

Seems like I saw an example of using good old curl to test a gRPC server amongst the hundreds of pages visited in the course of getting this thing to levitate.  B-but now grpcurl rises to the top of a quick search:

```bash
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
grpcurl -plaintext localhost:14586 list
 grpc.reflection.v1.ServerReflection
 grpc.reflection.v1alpha.ServerReflection
 photo.photoService
grpcurl -plaintext localhost:14586 list photo.photoService
 photo.photoService.addPhoto
 photo.photoService.deletePhoto
 photo.photoService.getPhotos
grpcurl -plaintext localhost:14586 photo.photoService/getPhotos
 {}
```

Empty server is empty!

Note that reflection is enabled in the server above so that things are nice and easy for grpcurl.

Now try out the client:

```bash
go run examples/demo/main.go
 found 48 photos
 >>> "http://tartu/photo/resized/PXL_20230707_101846985-4.png"
 >>> "http://tartu/photo/resized/PXL_20230707_110522671-4.png"
 ...
```

### Outstanding Red Team

At this point Protobuf is looking pretty good; steady rapid progress from an API specification to a working demo of server and client.
To be fair, I'm a bit of an old hand with Golang.

## Vue

### Generate JS

But first, more plugins:

```bash
curl -LO https://github.com/grpc/grpc-web/releases/download/1.4.2/protoc-gen-grpc-web-1.4.2-linux-x86_64
chmod 755 protoc-gen-grpc-web-1.4.2-linux-x86_64
mv protoc-gen-grpc-web-1.4.2-linux-x86_64 ~/mymymy/bin/protoc-gen-grpc-web

curl -LO https://github.com/protocolbuffers/protobuf-javascript/releases/download/v3.21.2/protobuf-javascript-3.21.2-linux-x86_64.tar.gz
tar zxvf protobuf-javascript-3.21.2-linux-x86_64.tar.gz
mv bin/protoc-gen-js ~/mymymy/bin/.
```

Ok, now generate JS!!

```bash
protoc --js_out=import_style=commonjs,binary:photo-client/src/ --grpc-web_out=import_style=commonjs,mode=grpcwebtext:photo-client/src/ photo.proto
ls photo-client/src/
 photo_grpc_web_pb.js
 photo_pb.js
```

### Import Generated Code

... in script of a Vue component ...

```js
import { getPhotoParams } from "photo_pb";
import { photoServiceClient } from "photo_grpc_web_pb";
```

### Clonk!!

The exports from the generated code are smooshed by Vite (rollup presumably) and the build process throws an error when imported into a component.
The internet says this is a recent problem with Vite and protobuf-javascript.
Switching to protobuf-ts (typescript) looks to be a popular choice .. after a few hours banging head/wall I'm ready to give it a try.

### Generate TS

'Nother plugin:

```bash
npm install @protobuf-ts/plugin -g
```

Ok, now generate TS!

```bash
mkdir pbts
cd pbts
protoc --ts_out . --proto_path ../. ../photo.proto
ls
 photo.client.ts
 photo.ts
```

### Import Generated Code

This time we're trying a Typescript flavored Vue component:

```js
<script setup lang="ts">
import { ref } from 'vue'

import { getPhotoParams } from './photo';
import { photoServiceClient } from './photo.client';
import { GrpcWebFetchTransport } from "@protobuf-ts/grpcweb-transport"

const client = new photoServiceClient(
  new GrpcWebFetchTransport({
    baseUrl: "http://192.168.88.117:8080"
  })
)

const getReply = () => {
  let request = getPhotoParams.fromJson({})

  client.getPhotos(
    request
  ).then(
    (res) => {
      let { response } = res
      console.log(response.photos)
    }
  )
}
</script>

<template>
  <div>
      <button class="button" @click="getReply">
        Send Messages to GRPC Server
      </button>
    </div>
</template>
```

And that makes all the difference!  The code builds and requests are sent as expected.

## Envoy Proxy

Now all we need to translate from grpc-web to gRPC with Envoy.
We'll run it in good olde docker.

Dockerfile:

```docker
FROM envoyproxy/envoy:v1.27.0
COPY envoy2b.yaml /etc/envoy.yaml
CMD /usr/local/bin/envoy -c /etc/envoy.yaml
```

Config file:

```yaml
admin:
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: echo_service }
              typed_per_filter_config:
                envoy.filters.http.cors:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.CorsPolicy
                  allow_origin_string_match:
                  - safe_regex:
                      regex: \*
                  allow_methods: "GET,POST,PUT,PATCH,DELETE,OPTIONS"
                  allow_headers: "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,Access-Control-Allow-Origin,x-grpc-web"
                  allow_credentials: true
                  max_age: "1728000"
          http_filters:
          - name: envoy.grpc_web
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
          - name: envoy.filters.http.cors
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.CorsPolicy
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
  - name: echo_service
    connect_timeout: 0.25s
    type: LOGICAL_DNS
    http2_protocol_options: {}
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: echo_service
      endpoints:
        - lb_endpoints:
          - endpoint:
              address:
                socket_address:
                  address: 192.168.88.117
                  port_value: 14586
```

I hacked on the config a little before this worked and doubtless there's some fat in there.
Anyway, let's put it to work:

```bash
docker build -t envoy .
 Sending build context to Docker daemon  11.26kB
 Step 1/3 : FROM envoyproxy/envoy:v1.27.0
 ---> 511f8ff2a1f9
 Step 2/3 : COPY envoy2b.yaml /etc/envoy.yaml
 ---> Using cache
 ---> 908ff1b74fdc
 Step 3/3 : CMD /usr/local/bin/envoy -c /etc/envoy.yaml
 ---> Using cache
 ---> 5e96c691a83b
 Successfully built 5e96c691a83b
 Successfully tagged envoy:latest
docker run  -p 8080:8080  envoy
 [2023-09-22 23:23:08.886][9][info][main] [source/server/server.cc:413] initializing epoch 0 (base id=0, hot restart version=11.104)
 [2023-09-22 23:23:08.886][9][info][main] [source/server/server.cc:415] statically linked extensions:
 ...
```

## Click the Button

On the web page coughed up by Vue and in the browser dev tools console we see:

```js
{id: 'asdfasdfasdf', largeName: 'http://tartu/photo/resized/PXL_20230707_101846985-4.png', thumbName: 'http://tartu/photo/resized/PXL_20230707_101846985-16.png', width: 3072, height: 4080, …}
{id: 'asdfasdfasdf', largeName: 'http://tartu/photo/resized/PXL_20230707_110522671-4.png', thumbName: 'http://tartu/photo/resized/PXL_20230707_110522671-16.png', width: 3072, height: 4080, …}
{id: 'asdfasdfasdf', largeName: 'http://tartu/photo/resized/PXL_20230707_110555944-4.png', thumbName: 'http://tartu/photo/resized/PXL_20230707_110555944-16.png', width: 3072, height: 4080, …}
{id: 'asdfasdfasdf', largeName: 'http://tartu/photo/resized/PXL_20230707_110559160-4.png', thumbName: 'http://tartu/photo/resized/PXL_20230707_110559160-16.png', width: 3072, height: 4080, …}
{id: 'asdfasdfasdf', largeName: 'http://tartu/photo/resized/PXL_20230707_111654739-4.png', thumbName: 'http://tartu/photo/resized/PXL_20230707_111654739-16.png', width: 4080, height: 3072, …}
```

Woot, it works!! : )

## Conclusion

I'm not convinced Protobuf is a good choice for a web application.
However it could be pretty cool in the datacenter.

### grpc-web and gRPC

This solution is hard to love and it shows.
The bug I ran into with Vite and protobuf-js is niche and had I been here a year ago, I don't think it would have come up.
On the other hand, it feels predictable.

I'm not feeling great about the Envoy dependency either.
Sure, it makes sense to proxy traffic from the internet, but translating protocols at multiple layers is more than an HTTP proxy.
I'm guessing that Envoy is _the_ product here.

### Golang Protobuf

This went pretty good and felt solid.
Chances are this would be a good experience with other languages and even with non-browser JS.
I'd like to give Protobuf another spin here, especially with lots of data and perhaps even a use case for streaming.

### API First

Which is to say working from an API specification.
This _is_ an appealing concept and I'll doubtless continue looking into implementations.
I do wonder though.
Could it be that it's easy to see the problems of ad-hoc JSON on the wire and less so those of a well constrained, shared specification?
Can these approaches be mixed?

### Conclusory Disclaimer

My opinions are my own; your mileage may vary, etc.
I am pushing myself to offer them in the hopes that the resulting texture adds to the usefulness of the post.

### Thanks

Hey you made it to the end!
Thank you for reading : )

