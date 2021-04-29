---
layout: default
title: gRPC-web: Using gRPC in Your Front-End Application
permalink: /grpc-web-frontend
---

# gRPC-web: Using gRPC in Your Front-End Application

At StackPulse, we use [gRPC](https://grpc.io/) as our one and only synchronous communication protocol. Microservices communicate with each other using gRPC, our external API is exposed via gRPC and our frontend application (written using [VueJS](https://vuejs.org/)) uses the gRPC protocol to communicate with our backend services. One of the main strengths of gRPC is the community and the language support. Given some proto files, you can generate a server and a client for most programming languages. In this article, I will explain how to communicate with your gRPC backend using the great [gRPC-web](https://github.com/grpc/grpc-web) OSS project.

## A quick reminder about our architecture

As you may remember from my previous [blog posts](https://grpcguide.com), we are using a microservice architecture at [StackPulse](https://stackpulse.io). When we initially started, each microservice had an external and internal API endpoint.

After working that way for several months, we realized it doesn’t work as well as we’d imagined. As a result, we decided to adopt the [API Gateway/Backend For Frontend](https://samnewman.io/patterns/architectural/bff/) approach.

![Current StackPulse architecture](/images/sp-arcg.png "Current StackPulse architecture")

I will publish a different blog post about this change in the future. However, the main pain point was data consolidation across several microservices—the dreaded JOINs over gRPC.

## Introducing gRPC Web

gRPC-web is a JavaScript implementation of [gRPC](https://grpc.io/) for browser clients. It gives you all the advantages of working with gRPC, such as efficient serialization, a simple IDL, and easy interface updating. A gRPC-web client connects to gRPC services via a special proxy, as shown below.

![gRPC Web Flow into Backend](/images/grpcweb/grpc-web-flow.png "gRPC Web Flow into Backend")

[Envoy](https://blog.envoyproxy.io/envoy-and-grpc-web-a-fresh-new-alternative-to-rest-6504ce7eb880) has built in support for this type of proxy.

[Here](https://github.com/improbable-eng/grpc-web/tree/master/go/grpcwebproxy) you can also find an example of the grpc-web proxy implementation in Golang. We use a similar implementation at StackPulse.

## Building gRPC Web clients

We generate gRPC clients for all our external APIs during the CI process. In the case of gRPC-web client, an npm package is generated and then published to the GitHub package repository. Then, JavaScript applications can consume this package using the `npm install` command.

![Example build of one of our services](/images/grpcweb/build.png "Example build of one of our services")

# Sample client/server example

Our proto interface

```proto
syntax = "proto3";

package smpl.time.api.v1;

option go_package = "github.com/kostyay/grpc-web-example/api/time/v1";

message GetCurrentTimeRequest {
}

message GetCurrentTimeResponse {
  string current_time = 1;
}

service TimeService {
  rpc GetCurrentTime(GetCurrentTimeRequest) returns (GetCurrentTimeResponse);
}
```

This is a really simple service. You send it `GetCurrentTimeRequest` and it returns `GetCurrentTimeResponse` containing the text representation of `time.Now().String()`.

### Generating clients and servers

In order to generate the clients and the servers for this proto file, you need to use the protoc command. Generating gRPC-web client and JavaScript definitions requires the `protoc-gen-grpc-web` plugin. You can get it [here](https://github.com/grpc/grpc-web) or use the pre-baked docker image `jfbrandhorst/grpc-web-generators` that contains all the tools needed to work with grpc-web.

This is the command I’m using to generate both the Go clients/servers and the JavaScript clients:


```
docker run \
    -v `pwd`/api:/api \
    -v `pwd`/time/goclient:/goclient \
    -v `pwd`/frontend/src/jsclient:/jsclient \
    jfbrandhorst/grpc-web-generators \
    protoc -I /api \
        --go_out=plugins=grpc,paths=source_relative:/goclient \
        --js_out=import_style=commonjs:/jsclient \
        --grpc-web_out=import_style=commonjs,mode=grpcwebtext:/jsclient \
/api/time/v1/time_service.proto
```

It will put the Go clients in `./time/goclient` and the JavaScript clients in `./frontend/src/jsclient`.

It’s worth noting that the client generator is also able to generate TypeScript code, which you can read more about in its [documentation](https://github.com/grpc/grpc-web#typescript-support).

### Backend

A really basic Go server which just listens on `0.0.0.0:8080`. It implements the `TimeServiceServer` interface and returns `time.Now().String()` for each request.

```go
package main

import (
	"context"
	"log"
	"net"
	"time"

	pb "github.com/kostyay/grpc-web-example/time/goclient/time/v1"
	"google.golang.org/grpc"
)

const (
	listenAddress = "0.0.0.0:9090"
)

type timeService struct {

}

func (t *timeService) GetCurrentTime(ctx context.Context, req *pb.GetCurrentTimeRequest) (*pb.GetCurrentTimeResponse, error) {
	log.Println("Got time request")
	return &pb.GetCurrentTimeResponse{CurrentTime: time.Now().String()}, nil
}

func main() {
	log.Printf("Time service starting on %s", listenAddress)
	lis, err := net.Listen("tcp", listenAddress)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterTimeServiceServer(s, &timeService{})

	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

### Frontend

Using gRPC-web in the frontend is pretty simple, as shown by the example below.


```javascript
// Import the client and the message definition
import { TimeServiceClient } from '../jsclient/time/v1/time_service_grpc_web_pb';
import { GetCurrentTimeRequest } from '../jsclient/time/v1/time_service_pb';

// Connect to the gprc-web server
const client = new TimeServiceClient("http://localhost:8080", null, null);
// This is a neat chrome extension that allows you to spy on grpc-web traffic just like you would on normal traffic.
// You can find it here: https://chrome.google.com/webstore/detail/grpc-web-developer-tools/ddamlpimmiapbcopeoifjfmoabdbfbjj?hl=en
const enableDevTools = window.__GRPCWEB_DEVTOOLS__ || (() => {});
enableDevTools([
  client,
]);

// Send getCurrentTime request
client.getCurrentTime(new GetCurrentTimeRequest(), {}, (err, response) => {
  // handle the response
  this.lastTimeResponse = response.getCurrentTime();
});
```

A small tip – I recommend enabling the [gRPC-web chrome extension](https://github.com/SafetyCulture/grpc-web-devtools). It’s a great way to inspect your gRPC traffic coming from and to the browser, just like you would with the Network Activity Inspector that is built into Chrome.

### Envoy configuration

Like I previously mentioned, gRPC-web needs a proxy to translate into gRPC. Envoy has native support for this, and the following configuration example for Envoy does exactly that.

```yaml
admin:
  access_log_path: /dev/stdout
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }

static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address: { address: 0.0.0.0, port_value: 8080 }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.config.filter.network.http_connection_manager.v2.HttpConnectionManager
                codec_type: auto
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route:
                            cluster: time_service
                            max_grpc_timeout: 0s
                      cors:
                        allow_origin_string_match:
                          - prefix: "*"
                        allow_methods: GET, PUT, DELETE, POST, OPTIONS
                        allow_headers: keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,custom-header-1,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout
                        max_age: "1728000"
                        expose_headers: custom-header-1,grpc-status,grpc-message
                http_filters:
                  - name: envoy.filters.http.grpc_web
                  - name: envoy.filters.http.cors
                  - name: envoy.filters.http.router
  clusters:
    - name: time_service
      connect_timeout: 0.25s
      type: logical_dns
      http2_protocol_options: {}
      lb_policy: round_robin
      load_assignment:
        cluster_name: cluster_0
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: time-server
                      port_value: 9090
```

## Final words

I hope this article will help you easily dive into gRPC-web. It is a great technology, especially if you are already using gRPC everywhere. We’re using it with great success in our frontend application. If you’re interested in learning more, you can get started with the source code for this article [here](https://github.com/kostyay/grpc-web-example).