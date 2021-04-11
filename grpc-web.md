---
layout: home
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