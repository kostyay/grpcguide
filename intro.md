# gRPC in practice, the introduction

![gRPC in / REST out](/images/intro_banner.png "gRPC in / REST out")

Each time I start working on a new product from scratch, I like to take some time and research the latest available technologies and patterns that companies are using. I always discover something new and improve on my previous experiences.

About 2 months ago I started working at StackPulse. When I joined there was not a single line of code written yet. We’ve had to decide on many different things. One of those things was choosing a “protocol” for communication both between our micro services and externally. We’ve had the option of sticking to what we knew worked, REST, or perhaps try something else. We’ve decided to give gRPC a try and use it as our main protocol for communication both internally and externally.

> gRPC is a modern open source high performance RPC framework that can run in any environment. It can efficiently connect services in and across data centers with pluggable support for load balancing, tracing, health checking and authentication. It is also applicable in last mile of distributed computing to connect devices, mobile applications and browsers to backend services. — grpc.io

While gRPC has been around for a long time now, we were still weary as if it may not be as simple to implement as REST. There are great tools around REST in all the programming languages and the frameworks. With gRPC you have to do quite a bit of plumbing to get everything to work. We also had to come up with original solutions to some challenges we faced along the way.

![Not the plumber you need to get gRPC to work in your project](/images/intro_plumber.png "Not the plumber you need to get gRPC to work in your project")

In this series of blog posts I will try to describe the challenges we faced and how we solved them.

## Let’s define our requirements

![Our architecture](/images/intro_diagrom.png "Our architecture")

1. We use the [microservice architecture](https://microservices.io/), each microservice is responsible for a specific business task.
2. Microservices may have internal and external APIs.
3. Internal APIs are invoked by other microservices. They should not be publicly exposed. They don’t require authentication and are not accessible outside the microservice inner network.
4. External APIs can be invoked by authenticated users or authenticated API clients. They are public and well documented.
5. We would like to have a unified and expressive way to define these APIs, preferably with some kind of a language. It should be simple for us to do it while maintaining backward compatibility and predefined set of internally defined development style and conventions. Anyone reading the service definition should be able to understand what the service does and how to interact with it.
6. We would like to provide easy API access to external APIs. Writing clients for these APIs in different programming languages should be simple.
7. It should be possible to auto-generate client and server code for both internal and external APIs
8. Our JavaScript browser based frontend should be able to communicate with the microservices’ external API.

Basically gRPC gives us all that and more.

## But wait, why not just use REST API?

I’ve used REST API in all the previous products I’ve built. It’s very common and simple to use, definitely battle tested. There are clients for it in all programming languages. There is an expressive way to define them called OpenAPI (the swagger files).

We could as well just chosen REST API, the ecosystem for it is huge and pretty much everybody worked or is still working with it.

Yet, despite that we’ve decided to go with gRPC, for the following reasons:

1. Its a binary protocol that uses [protocol buffers](https://developers.google.com/protocol-buffers) for serialization and runs over HTTP/2. This should provide better performance compared to REST API.
2. It supports streams out of the box.
3. The client and the server code it generates is very good and reliable. There are good tools available for this, specifically for Golang which we use and love.
4. The [protocol buffer](https://developers.google.com/protocol-buffers) language used to define messages is strongly typed and very straight to the point. It is backward compatible by design. There are excellent linting tools available for it.

You can read more about the comparison of gRPC and REST API in [this article](https://code.tutsplus.com/tutorials/rest-vs-grpc-battle-of-the-apis--cms-30711), you can find some performance comparison in [this article](https://medium.com/@EmperorRXF/evaluating-performance-of-rest-vs-grpc-1b8bdf0b22da).

In the next article I will go in detail about on the implementation, so get ready to get your hands dirty.