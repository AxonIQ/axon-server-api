# AxonServer API Specification

This module contains the protobuf definitions of the services and messages provided by
AxonServer.

The API definitions are structured around 5 files:
- `common.proto` - contains messages commonly used throughout the services
- `control.proto` - contains services and messages to monitor and control application components
- `command.proto` - contains the service and message definitions specific to the dispatching and handling of commands.
- `event.proto` - contains services and messages for publishing and consuming events
- `query.proto` - contains services and messages for dispatching and handling queries

## Building a client
### Generating stubs

Based on the protobuf files, stubs can be generated for several languages and platforms. Visit the [gRPC quick start](https://www.grpc.io/docs/quickstart/) page for your language. 

### Attaching security headers

In all communication to AxonServer Enterprise, the client will need to provide the name of the Context it belongs to. 
This context is added as a header to each request. The key of the header is `AxonIQ-Context`, the value is the 
ascii-encoded name of the Context.

If security is enabled on AxonServer, a security header also needs to attached to each request. The key of that header is
`AxonIQ-Access-Token`, and the value is the ascii-encoded token which identifies the client.

### Connecting to AxonServer

In a cluster of AxonServer nodes, not every node has the same role. That means that not every node may be suitable
to serve as a connection counterpart to every application. Therefore, connecting to AxonServer is a
two-step process.

The first step is to establish a connection with an "admin" node. These nodes have detailed information about the layout
of the entire cluster. Typically, several nodes act as "admin" node. Your client will need to connect to just one
of them.

The first call to the Admin node is the `PlatformService.GetPlatformServer`, defined in the `control.proto` file. In the request,
you pass information about the client trying to connect. The tags are optional, and may provide hints to the AxonServer
cluster about connection preferences the application may have. The result provides connection details of the AxonServer
node the client should defer its connection to. In case the `same_connection` flag is `true`, the client may use the 
existing connection for further communication. Otherwise, the existing connection must be closed, and a connection must 
be set up using the provided hostname and port.

Well-behaved clients will always set up a two-way instruction stream using the `PlatformService.OpenStream` rpc. This is
a two-way streaming rpc, which allows AxonServer to provide instructions to the client, for example to notify it of a 
node shutting down, or a request to connect to another instance, to better balance connections across the cluster.

Finally, depending on the responsibilities of the client, it can register command handlers, query handler, open event 
streams, and send messages.

### Flow control

AxonServer uses flow-control to ensure clients are not flooded with more messages than they can handle. For historic 
reasons, flow control is explicitly implemented on the RPC level. Whenever, messages are streamed from AxonServer to a 
client, the client must indicate how many messages the client can ingest before the Server node needs to wait for more 
"permits" to be allocated.

Permits are cumulative. When opening a connection to subscribe a command handler, for example, and providing 500 
permits, AxonServer may send up to 500 messages without waiting in between. If, after receiving 100 messages, 500 more 
permits are sent, the total available permits for AxonServer is 900.

It is highly recommended to only refresh permits after messages are consumed from any internal buffers. This helps 
ensure these buffers never fill up beyond expected proportions.

When AxonServer has messages to send to a node that doesn't have any permits available, AxonServer will buffer them on 
the server side. However, those buffers are also limited. Once they fill up, depending on AxonServer settings, either 
new messages will be rejected (sending errors to the clients dispatching them), or AxonServer will start adjusting its 
routing properties to reduce the load on that specific client.
 
### Forward and backward compatibility considerations

Protobuf messages, by design, account for forward and backwards compatibility. The serialized form of messages contain 
just enough information to make sure a client can successfully consume the message. However, to assign meaning to these
elements, the proto files are required.

AxonServer takes backwards and forwards compatibility into account, too. The binary format of messages will be 
compatible with future versions of AxonServer. Typically, elements may be deprecated in minor versions, but will only 
be removed in the next major release of an API. 

However, new versions of AxonServer may provide instructions to clients that a certain version of the client does not
understand. In that case, the client may simply ignore these instructions. Similarly, a client must not assume that the
server understands all instructions, as a client may be talking to an older version of AxonServer, too. 
