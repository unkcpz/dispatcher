# Dispatcher

## Reminders

Q: major difference and essential parts compare to neovim or other plugin system are: 
- the EOSC data common is for redirect. It is still an open question which might be out of scope to get analysis result back to the EOSC system. 
- the tool is online services that can change without notify the registry therefore fail the actual usage. (see if https://github.com/apps/renovate can be used for registered tool).

## RFC001: plugin system

The plugin system is an abstraction for 3rd-party tool provider to register their tool into the EOSC data commons. 
The goal will be:

- tool provider files a request (declaritive request) into a tool store.
- tool store moderator reviews the tool (security, intellecture properties, tool availability, tool quality. etc). 
- Once the tool approved, it appears in the tool store so the dataset can be inspected and played by the tool.
- the plugin is added as service without the needs of changing any EOSC infrastructure, and the plugin is added by adding API support along the protocol and then register in the tool store.

The tool when registred, it should declare with following included information so EOSC knows what it is and how to communicate with the tool:

- what type is the tool? 
- Do this tool has dependencies of other tool? 
- Do this tool require external resources?
- The API endpoint of the tool to communicate with.
- how to send the dataset to the tool?
- can tool able to handle to fetch the data themselves?

### Architecture

- the rpc client is locate in the EOSC deployment.
- the tools are services hosted by tool provider and deployed as the rpc services which conform with the protocol.

### RPC or dependendency injection?

- the remote plugins in neovim are typically using rpc to communicate.
- aiida is simply a dependendency injection pattern.

### What EOSC provide

- SDK for tool provider to implement the plugin support?
- or the restapi to talk with?

### Registry examplers 

The tools need to be registered so the system can lits and load them.

- https://github.com/mason-org/mason-registry
- https://github.com/aiidateam/aiida-registry

### Open questions

- Is the resources should be described as tools? (TBD in WP5)
- how much efforts are acceptable for a 3rd-party tool to support to be added into EOSC? (who do the tool registry? who implement the plugin? will it require the change of API of original tool? Is implementation the adaptor to conform the protocol hard?)
- how to validate the availability and the health of the tools? It must be automatic or get report from users. (the call is made through EOSC and error status are captured in EOSC side.)

### Potential targeting 3rd-party VREs 

- Renku
- AiiDAlab

## RFC002: dispatcher as server service in EOSC system

dispatcher is accessable as a server service.
It can be communicated using gRPC, and the protobuf for the interface is implemented in this repo. 

There are two strong reasons to use gRPC over restapi:

1. the VRE information is send as a ro-crate which might even contains zip data that is much efficient in HTTPv2.
2. the client (matchmaker UI server) need updated information without keep on polling for the states update. The gRPC bilateral streaming fit well.

## RFC003: EOSC tool protocol

The potential lifecycle between dispatcher and a tool to be launched and run is:

1. Dispatcher launches tool process
2. Dispatcher connects via TCP
3. `tool.handshake`
4. `tool.start`
5. Dispatcher polls `tool.status`
6. Tool reaches terminal state
7. Tool exits
8. Dispatcher cleans up

- use jsonrpc for transfer the messages, https://www.jsonrpc.org/specification
- protocol spec define using typescript just as dap and lsp:
    - https://microsoft.github.io/debug-adapter-protocol/specification
    - https://microsoft.github.io/language-server-protocol/specification

### specification

- The most basic endpoints will be:
    - launch this tool for me.
    - open a file send from the matchmaker side.
    - what is the state of an operation.
    - where I can find the session to redirect to.

The dispatcher talk to tools by following the protocol so the tools can be added without knowing the detail of the dispatcher implementation.
The dispatcher in this role is the client using json-rpc and the tools interface should be the server side that understand the json-rpc and can send response back in json-rpc to dispatcher.

Design goals for the tool protocol spec:

- Stateless RPCs where possible
- dispatcher owns lifecycle & IDs
- Tools don’t know about gRPC
- Easy to implement in Rust / Python / anything
- json-rpc 2.0 compliant

Here are minimal protocol interfaces that forming a PoC.

#### Handshake

Do handshake first do the sanity check or the compalitibility check of the tool with its declaration.
The version information is checked from day one.
It allows dispatcher to reject incompatible tools.
This requirement reveal that the dispatcher need to talk to the tool registry.
But the tool registry can be a service, but might be just a curated file (or a dir with files of registered tool metadata.) contains all the current tool information.

dispatche -> tool:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tool.handshake",
  "params": {}
}
```

Tool -> Dispatcher

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "name": "example-tool",
    "version": "1.0.0",
    "protocol_version": 1
  }
}
```

#### launch / start

Dispatcher -> tool

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tool.start",
  "params": {
    "tool_id": "abc-123",
    "parameters": {
      "input_file": "data.xyz",
      "iterations": 1000
    }
  }
}
```


tool -> dispatcher

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": "ok"
}
```


Rules:
- tool must not generate its own ID, but through the tool registry?
- tool must associate all state with `tool_id`
- execution may start async after returning "ok".

#### Pooling states

dispatcher -> tool

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tool.status",
  "params": {
    "tool_id": "abc-123"
  }
}
```

Tool -> dispatcher

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "state": "running",
    "message": "step 42 / 100"
  }
}
```

The allowed state values are:

```
pending | running | succeeded | failed | cancelled
```

#### cancellation

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "tool.cancel",
  "params": {
    "tool_id": "abc-123"
  }
}
```

tool -> dispatcher

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": "ok"
}
```

#### Errors ( try to be JSON-RPC standard)

failure on start:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "error": {
    "code": 1001,
    "message": "Invalid parameter: iterations must be > 0"
  }
}
```

potential error code ranges:
- 1000–1099: invalid input
- 1100–1199: runtime failure
- 1200–1299: internal tool error

