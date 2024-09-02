---
layout: default
title: "gRPC: 5 Years Later, Is It Still Worth It?"
permalink: /is-grpc-still-worth-it-in-2024/
---
# gRPC: 5 Years Later, Is It Still Worth It?

![gRPC: 5 Years Later, Is It Still Worth It?](/images/5years/header.png "gRPC: 5 Years Later, Is It Still Worth It?")

It’s hard to believe that nearly five years have passed since I joined [Torq](https://torq.io). It seems like only yesterday that the founding team gathered in a temporary office — a repurposed TV show set — just four weeks before the COVID-19 pandemic began.

During those early days, we were making crucial decisions about the technology stack that would form the foundation of Torq. One decision stands out clearly in my memory: our unanimous agreement to avoid using OpenAPI/Swagger with Go in our future projects. This choice was born from our past collective experiences.

Some of us had worked together before at Luminate Security, where we had a tough time using [OpenAPI](https://www.openapis.org/) (aka Swagger) with Go. At [Luminate](https://en.globes.co.il/en/article-symantec-buys-israeli-cybersecurity-co-luminate-1001272925), all our microservices and frontend communication used REST with Swagger files. The problem was that the Go tools for making clients and servers from Swagger were not good back then. This meant every developer ended up making their own clients in different ways. We didn’t want to face the same issues again. That’s why we decided to fully commit to using gRPC and Protobuf for our new project.

We tried a [few different ways](/directory-structure-linting) to set up our proto files and wondered if we needed an API Gateway. After some time, we settled on this setup:
![gRPC in Frontend and Backend](/images/5years/diagram.webp "gRPC in Frontend and Backend")

Looking back, choosing gRPC as our sole communication protocol was an excellent decision. This choice has brought us several advantages:

1. It helped us maintain backward compatibility as our system evolved.
1. We could enforce standards through linting, ensuring consistent code quality.
1. Both clients and servers use the same generated code, reducing discrepancies.

These factors have made it easier for our engineers to work across different microservices without unexpected issues. Additionally, our shared codebase provides built-in benefits, including standardized authentication, authorization, and observability middleware. This approach has significantly streamlined our development process and improved overall system reliability.

Even today, if I were starting a new project from scratch, I would undoubtedly choose gRPC as my primary communication protocol. The gRPC ecosystem has grown and improved considerably over the past few years, largely due to the outstanding contributions of the [buf.build](https://buf.build) team. Their work has significantly enhanced the tooling and overall experience of working with gRPC, making it an even more attractive option for modern software development projects.

The project introduced improvements in many aspects, but most importantly in the following ones.

![buf.build](/images/5years/buf.webp "buf.build")

# Code Generation
Five years ago, when we began our journey, the [protoc](https://grpc.io/docs/protoc-installation/) compiler was the only available tool for working with gRPC and Protocol Buffers. This tool, while powerful, has a steep learning curve and a user experience typical of traditional Linux command-line utilities — not particularly user-friendly.

To improve this situation, I invested several weeks in creating a Docker container wrapper for it. This container encapsulated protoc along with all the code generation plugins we used internally and some bash scripts gluing everything together. Our primary goal was to enhance the developer experience for our engineering team. We wanted to eliminate the need for each engineer to individually install protoc and multiple plugins and to make generating Go or TypeScript assets a single command a developer could execute.

This approach ensured that everyone on the team was using identical versions of these tools throughout their development process. It significantly reduced setup time and potential version conflicts, allowing our engineers to focus more on actual development rather than tooling configuration.

![protoc vs buf generate](/images/5years/commands.webp "protoc vs buf generate")

Today, things are much easier thanks to the `buf generate` command. To get a project up and running, you now only need to create two files in your project directory:
1. [buf.yaml](https://buf.build/docs/configuration/v2/buf-yaml) — a configuration file that defines your proto directory layout and the linting rules you want to impose on them
1. [buf.gen.yaml](https://buf.build/docs/configuration/v2/buf-gen-yaml) — a configuration file used by the buf generate command to generate integration code for the languages of your choice

This simple setup achieves the same results as our previous Docker image approach. It allows developers to compile proto files along with all their dependencies, use many different code generators, or incorporate custom plugins. The best part is that developers can do all this without having to install or configure anything on their local machines. This streamlined process significantly improves efficiency and reduces potential setup issues.

# Module Dependencies
![Typical module dependency scenario (you should probably avoid)](/images/5years/module-deps.png "Typical module dependency scenario (you should probably avoid)")

When we started with gRPC/protos, a major challenge was deciding whether to reuse proto messages from other microservices or redefine them in each service. While generally it’s __best to avoid__ reuse, exceptions exist mainly for infrastructural messages.

Back then, there was no best practice for sharing proto files across multiple repositories. While I prefer using multiple repositories for code organization, monorepos (which I understand is Google’s preferred approach) had an advantage here as you could just reference other service’s proto file in another directory. To enable code sharing in a multi-repo setup, we had to rely on a tool called [protodep](https://github.com/stormcat24/protodep). This simple dependency manager for proto files essentially clones files from other microservices’ GitHub repositories and places them in a directory alongside your own proto files (think of go vendor few years back). I’ve written a detailed explanation of protodep in a previous blog post.

Fortunately, these complications are now a thing of the past, thanks to the [Buf Schema Registry (BSR)](https://buf.build/product/bsr). BSR serves as a central hub for managing and evolving Protobuf APIs. This centralized system offers several key benefits:
1. It helps maintain compatibility across different versions of your APIs.
1. It simplifies dependency management.
1. It ensures clients can consume APIs reliably and efficiently by auto-generating client code for different programming languages.
1. It automatically generates interactive documentation for your gRPC services and proto messages.

Think of BSR as a comprehensive build system for your proto files. One of its most valuable functions is the ability to manage proto module dependencies in a straightforward manner. This feature alone addresses many of the challenges we previously faced with proto file sharing and version control in multi-repo environments.

```yaml
# buf.yaml example with external module dependencies
version: v2
# ....
# Dependencies shared by all modules in the workspace. Must be modules hosted in the Buf Schema Registry.
# The resolution of these dependencies is stored in the buf.lock file.
deps:
  - buf.build/acme/paymentapis # The latest accepted commit.
  - buf.build/acme/pkg:47b927cbb41c4fdea1292bafadb8976f # The '47b927cbb41c4fdea1292bafadb8976f' commit.
  - buf.build/googleapis/googleapis:v1beta1.1.0 # The 'v1beta1.1.0' label.
```

# Using gRPC in your Frontend

When I tell people we use gRPC-web in our frontend code, they often look surprised. I haven’t met other companies using it, but since there are tools for it, I guess some do.

Using gRPC-web comes with several problems (or challenges).

First, it uses a binary format that you can’t easily check with your browser’s Network Inspector. Everything is encoded in base64, which makes frontend debugging work harder, and can make your frontend engineers unhappy they can’t use the tools they are used to thus slowing them down. There’s a Chrome extension called [gRPC-Web Developer Tools](https://chromewebstore.google.com/detail/grpc-web-developer-tools/kanmilmfkjnoladbbamlclhccicldjaj?hl=en) that helps with checking network traffic, but for my taste it’s not as good as Chrome’s built-in inspector.

Also, when using gRPC-web, you need a proxy on your backend server to change the gRPC-web format into regular gRPC so your services can use it. You can do this with Envoy, but not everyone uses it. If you use nginx, you’ll have to add the translation yourself in your service. Doing this might cause security problems because you might not know all the protocols it supports, and your security checks might not work as you expect. For example gRPC-Web supports [websockets](https://github.com/improbable-eng/grpc-web/blob/master/client/grpc-web/docs/websocket.md) for client side streaming, but the middleware you configured for your HTTP handlers may not be invoked there.

Because gRPC-web is implemented as POST method you will not be able to support Caching for your calls.

Another issue (or maybe a good thing) with gRPC-web is that when security teams try to test your app, they might not have their usual tools for testing gRPC-web. This could be seen as security through obscurity.

Thankfully there is an alternative now called [connectrpc](https://connectrpc.com/) (from the [buf.build](https://buf.build) team). It comes as a set of protoc plugins to generate high quality TypeScript clients and despite natively supporting the grpc-web transfer protocol it also supports its own json based protocol which also supports caching.

![connectrpc](/images/5years/connectrpc.webp "connectrpc JSON protocol")

# Service Mesh
As you may, or may not know Kubernetes’ default load balancing [is not effective out of the box with gRPC](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/).

 This is because gRPC is built on HTTP/2, and HTTP/2 is designed to have a single long-lived TCP connection, across which all requests are multiplexed — meaning multiple requests can be active on the same connection at any point in time. Normally, this is great, as it reduces the overhead of connection management. However, it also means that (as you might imagine) connection-level balancing isn’t very useful. Once the connection is established, there’s no more balancing to be done. All requests will get pinned to a single destination pod, as shown below ([source](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/)):
 ![HTTP2 connection reuse](/images/5years/linkerd.webp "HTTP2 connection reuse")

In order to overcome this, and due it’s superb observability and security capabilities we have been [linkerd](https://linkerd.io/2-edge/features/load-balancing/) users since day one.

Besides proper load balancing for gRPC it provides multiple advantages:
1. Built in “top-line” metrics for all the meshed services such as request volume, success rate, and latency distributions.
1. Zero configuration mTLS ensuring data flowing between pods is encrypted
1. Traffic authorization policy — you can restrict communication to a particular service (or HTTP route on a service) to only come from certain other services

# So would I use gRPC again?
__Absolutely__, my protocol of choice remains gRPC with the following stack:

1. [buf.build](https://buf.build) to compile generate protos for the programming languages or if using BSR you don’t even have to compile them yourself, just use the code generation features of BSR.
2. [BSR](https://buf.build/product/bsr) for module dependency management, linting, backward compatibility validation and API documentation management
3. Service To Service protocol — gRPC over linkerd
4. Frontend to backend — [connectrpc](https://buf.build/product/bsr) with JSON payloads