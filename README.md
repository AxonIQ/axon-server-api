# Axon Server API

This module contains the protobuf definitions of the services and messages provided by Axon Server.
It is published as the `io.axoniq:axon-server-api` artifact on Maven Central, which bundles the `.proto` files.

The API definitions are structured around the following files:

- `admin.proto` - contains services and messages to monitor and control Axon Server
- `common.proto` - contains messages commonly used throughout the services
- `control.proto` - contains services and messages to monitor and control application components
- `command.proto` - contains the service and message definitions specific to the dispatching and handling of commands.
- `event.proto` - contains services and messages for publishing and consuming events
- `dcb.proto` - contains services and messages for the Dynamic Consistency Boundary (DCB) event store, snapshot store
  and event scheduler
- `query.proto` - contains services and messages for dispatching and handling queries
- `persistent-streams.proto` - contains services and messages for creating, managing and consuming persistent streams
- `license.proto` - contains the service and messages for retrieving Axon Server license information

## About this repository

This repository contains the Axon Server API specification, published under a commercial, source-available license.
We publish it here for transparency, so you can inspect the API you're building against.

This is not an open-source project in the OSS sense:

- This repository does not accept external pull requests, and we do not maintain a public contribution process for it.
- Being able to read the source does not grant open-source usage rights. Under the license terms, you may modify the
  code for internal use only. You may not distribute the original or modified versions of this software, and you may not
  distribute the source code or any derivative works to third parties.
- If you're looking to contribute code to the Axon ecosystem,
  see [AxonIQ/AxonFramework](https://github.com/AxonIQ/AxonFramework), which remains our open-source project and does
  accept contributions.

The [LICENSE.txt](./LICENSE.txt) contains a short reference that the Axon Server API is licensed under
the [Axoniq Terms of Service](https://www.axoniq.io/legal/terms-of-service) for the controlling legal terms. Questions
about licensing, permitted use, or usage rights should go to your Axoniq account contact, not GitHub issues.

## Building a client

### Generating stubs

Based on the protobuf files, stubs can be generated for several languages and platforms. Visit
the [gRPC quick start](https://www.grpc.io/docs/quickstart/) page for your language.

### Attaching security headers

In all communication to Axon Server, the client will need to provide the name of the Context it belongs to. 
This context is added as a header to each request. The key of the header is `AxonIQ-Context`, the value is the 
ascii-encoded name of the Context.

If security is enabled on Axon Server, a security header also needs to attached to each request. The key of that header is
`AxonIQ-Access-Token`, and the value is the ascii-encoded token which identifies the client.

### Connecting to Axon Server

In a cluster of Axon Server nodes, not every node has the same role. That means that not every node may be suitable
to serve as a connection counterpart to every application. Therefore, connecting to Axon Server is a
two-step process.

The first step is to establish a connection with an "admin" node. These nodes have detailed information about the layout
of the entire cluster. Typically, several nodes act as "admin" node. Your client will need to connect to just one
of them.

The first call to the Admin node is the `PlatformService.GetPlatformServer`, defined in the `control.proto` file. In the request,
you pass information about the client trying to connect. The tags are optional, and may provide hints to the Axon Server
cluster about connection preferences the application may have. The result provides connection details of the Axon Server
node the client should defer its connection to. In case the `same_connection` flag is `true`, the client may use the 
existing connection for further communication. Otherwise, the existing connection must be closed, and a connection must 
be set up using the provided hostname and port.

Well-behaved clients will always set up a two-way instruction stream using the `PlatformService.OpenStream` rpc. This is
a two-way streaming rpc, which allows Axon Server to provide instructions to the client, for example to notify it of a 
node shutting down, or a request to connect to another instance, to better balance connections across the cluster.

Finally, depending on the responsibilities of the client, it can register command handlers, query handler, open event 
streams, and send messages.

### Flow control

Axon Server uses flow-control to ensure clients are not flooded with more messages than they can handle. For historic 
reasons, flow control is explicitly implemented on the RPC level. Whenever, messages are streamed from Axon Server to a 
client, the client must indicate how many messages the client can ingest before the Server node needs to wait for more 
"permits" to be allocated.

Permits are cumulative. When opening a connection to subscribe a command handler, for example, and providing 500 
permits, Axon Server may send up to 500 messages without waiting in between. If, after receiving 100 messages, 500 more 
permits are sent, the total available permits for Axon Server is 900.

It is highly recommended to only refresh permits after messages are consumed from any internal buffers. This helps 
ensure these buffers never fill up beyond expected proportions.

When Axon Server has messages to send to a node that doesn't have any permits available, Axon Server will buffer them on 
the server side. However, those buffers are also limited. Once they fill up, depending on Axon Server settings, either 
new messages will be rejected (sending errors to the clients dispatching them), or Axon Server will start adjusting its 
routing properties to reduce the load on that specific client.
 
### Forward and backward compatibility considerations

Protobuf messages, by design, account for forward and backwards compatibility. The serialized form of messages contain 
just enough information to make sure a client can successfully consume the message. However, to assign meaning to these
elements, the proto files are required.

Axon Server takes backwards and forwards compatibility into account, too. The binary format of messages will be 
compatible with future versions of Axon Server. Typically, elements may be deprecated in minor versions, but will only 
be removed in the next major release of an API. 

However, new versions of Axon Server may provide instructions to clients that a certain version of the client does not
understand. In that case, the client may simply ignore these instructions. Similarly, a client must not assume that the
server understands all instructions, as a client may be talking to an older version of Axon Server, too.

## Receiving help

Are you having trouble using this API?
Know that we want to help you out the best we can!
There are a couple of things to consider when you're traversing anything Axon:

* Checking the [reference guide](https://docs.axoniq.io) should be your first stop.
* When the reference guide does not cover your predicament, we would greatly appreciate it if you could file
  a [documentation issue](https://github.com/AxonIQ/axon-server-api/issues) for it.
* Our [forum](https://discuss.axoniq.io/) provides a space to communicate with the Axon community to help you out.
  Axoniq developers will help you out on a best-effort basis.
  And if you know how to help someone else, we greatly appreciate your contributions!
* We also monitor Stack Overflow for any question tagged with [**axon
  **](https://stackoverflow.com/questions/tagged/axon).
  Similarly to the forum, Axoniq developers help out on a best-effort basis.

## Reporting bugs

Found a bug? Open an issue here with:

- Steps to reproduce
- The Axon Server API version
- Relevant logs

We read every issue. Response times follow your support tier, not GitHub's issue queue.

## Feature requests

Feel free to open an issue tagged `enhancement`. We track these, but roadmap decisions are made internally and we can't
commit to timelines here.

## Reporting security issues

Found a security vulnerability? Do not open a public issue.
Email [security-disclosure@axoniq.io](mailto:security-disclosure@axoniq.io) directly so we can assess and patch before
disclosure.

## Code of conduct

Participation in this repository (issues, discussions, etc.) is governed by our [Code of Conduct](./CODE_OF_CONDUCT.md).

---

Copyright (c) Axoniq B.V. All rights reserved. Licensed under
the [Axoniq Terms of Service](https://www.axoniq.io/legal/terms-of-service).
