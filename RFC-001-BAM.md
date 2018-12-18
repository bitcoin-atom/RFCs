# BAM! - Bidirectional Application Messaging

- [Description](#description)
- [Status](#status)
- [Motivation & Requirements](#motivation--requirements)
- [Evaluation of existing protocols](#evaluation-of-existing-protocols)
    - [HTTP(2)](#http2)
    - [WebSockets](#websockets)
    - [A profile for BEEP](#a-profile-for-beep)
    - [gRPC](#grpc)
- [Introducing **BAM!**](#introducing-bam)
    - [Frames](#frames)
        - [Type](#type)
        - [Id](#id)
    - [Different frame types](#different-frame-types)
        - [Notification](#notification)
        - [Request / Response](#request--response)
            - [Structure](#structure)
            - [Optional fields](#optional-fields)
            - [Type](#type)
            - [Status](#status)
            - [Application data](#application-data)
            - [Headers](#headers)
            - [Status code families](#status-code-families)
                - [Successful responses (OK00-OK99)](#successful-responses-ok00-ok99)
                - [Sender errors (SE00-SE99)](#sender-errors-se00-se99)
                - [Receiver errors (RE00-RE99)](#receiver-errors-re00-re99)
    - [Headers](#headers)
    - [JSON Encoding](#json-encoding)
        - [Frames](#frames)
        - [Encoding of the individual frame types](#encoding-of-the-individual-frame-types)
            - [Request](#request)
            - [Response](#response)
        - [Header](#header)
            - [Representation](#representation)
            - [Default values](#default-values)
            - [Multiple values](#multiple-values)
            - [Compact representation](#compact-representation)
            - [Data types](#data-types)
        - [Naming conventions](#naming-conventions)
- [References](#references)

## Description

This RFC describes BAM!: **B**i-directional **A**pplication **M**essaging. It is a domain-agnostic and self-descriptive protocol and potentially supports multiple encodings.

## Status

```
RFC-Number: 001
Status: Draft
```

## Motivation & Requirements 

In COMIT, nodes need to exchange all kinds of information in a peer-to-peer manner. As COMIT is an inherently distributed system, the network will eventually become heterogeneous, with an array of implementations, varying in languages and versions. Given that, we need a transport protocol that fulfills the following requirements:

- **Different kinds of messages:** The protocol needs to support requests (response expected) and one-way transmissions (no response).
- **Self-descriptive messages:** Because of the heterogeneous nature, the protocol needs to allow the introduction of new features without breaking older versions. Self-descriptive messages allow backward compatibility because older implementations can still parse newer messages and continue even if they don't fully understand its content. (Spoiler: There are also ways to force backward incompatibility.)
- **Dynamic/Polymorphic messages:** Similar to HTTP, the messages sent to the other party don't always take the exact same form although they convey the same meaning. GET requests for example always describe the intent of retrieving a resource. The concrete structure of two GET requests might differ though. Similar to that, we need to send messages between COMIT nodes that communicate a certain intent (like initiate an atomic swap) but might take different structure (for example, different information  needs to be exchanged based on the ledger).
- **Peer-to-peer:** In a communication between two nodes, not always the same one initiates, e.g. both nodes need to be able to actively send messages to each other. Contrary to this requirement, HTTP for example demands that the client always initiates the connection/communication. In addition, it should be possible for nodes to work *behind* a NAT. This means, nodes should only use *one* physical connection to talk in both directions.

## Evaluation of existing protocols

We looked at several existing application protocols. In particular, we evaluated the following:

- HTTP(2)
- WebSockets
- A profile for BEEP (RFC 3080)
- gRPC and protobuf

### HTTP(2)

HTTP by definition is a client-server protocol. Unfortunately, this almost rules it out right from the beginning. With HTTP2 though, the protocol introduces so-called `Frames` which, among other things, allow the often cited PUSH-functionality of HTTP2. One way to use HTTP2 would be to modify the protocol slightly, so that each party could send `Frames` at any time: Means also the "server" could send request frames to the client and vice versa.

We decided against this approach because we feared it might falsely suggest compatibility with HTTP2 software, although this compatibility is not given. Using HTTP (1 or 2) the intended way doesn't work because it only supports uni-directional communication.

### WebSockets

WebSockets are the Web's solution for bidirectional communication. Although it would support the peer-to-peer requirement, WebSockets have a very low abstraction level, as they don't specify anything about the actual message that is sent over. Instead, the websocket spec mostly deals with things that are related to the Web (what a surprise!) like protection of shared infrastructure through masking of the actual payload or upgrades of HTTP connection to a web socket connection.

While WebSockets could have been a solution, they actually don't provide any benefit over plain sockets for us. In order to keep the complexity low, we decided against it.

### A profile for BEEP

BEEP is a fully bidirectional application protocol and looks very much like what we needed at first sight. It covers a wide range of requirements and has many features. Unfortunately, it never seemed to get traction and thus, library support is very limited. For Rust, we would have had to write our own implementation.

Although BEEP would fulfill all the requirements, there is no implementation for it. We could create and implementation but that would be quite a lot of work because BEEP covers much more than we need.

### gRPC

Whilst gRPC is a popular way of defining messaging between applications, it doesn't really fit our requirements: It doesn't work in a peer-to-peer context as it is centered around the idea of Client-Server. The messages are neither self-descriptive nor dynamic: In order to parse a message, a party needs to know the message format upfront. This is usually achieved through compiling `protobuf` files into source code. The messages are also not dynamic as their structure needs to be known at compile-time.

Given that, we decided against gRPC.

## Introducing **BAM!**

Given the above evaluation, we are left with defining our own protocol that implements the stated requirements. It incorporates ideas from several other protocols and combines them for our usecase. In particular, it borrows ideas from the following protocols:

- BEEP
- HTTP2
- Lightning network

The main concept in this protocol is called a `FRAME`.

### Frames

Frames identify themselves through a `type` and an `id` and carry a `payload`.

#### Type

The `type` defines how the `payload` of a frame is encoded. Types also define the certain semantics of a frame, for example, whether an implementation should wait for a response to this frame or not.

The following types are allowed:

- REQUEST
- RESPONSE

The following types are reserved for future use:

- NOTIFICATION

#### Id

The `id` of a frame allows nodes to associate frames with each other. Each node MUST keep track of the `id` it used on its own. Nodes SHOULD start with the value `1` for this id. The id MUST fit into an unsigned 32-bit integer. This means the maximum allowed value is `4294967295`.

Ids MUST NOT be reused for subsequent frames.

A node should only assign `id`s to frames that are out-going and self-initiated, for example `REQUEST` frames. `RESPONSE` frames MUST have the same `id` as the `REQUEST` frame they belong to.

Ids MAY be skipped but MUST be ascending.

### Different frame types

#### Notification

> `NOTIFICATION` is a reserved type and may be specified at a later point.

#### Request / Response

A request is a type of message that implies an answer. Nodes MUST be prepared to receive a frame of type `RESPONSE` with the id used in the `REQUEST` frame.

##### Structure

A frame of type `REQUEST` carries the following payload:

- Type
- Application data
- Headers
- Body

whereas a frame of type `RESPONSE` looks like this:

- Status
- Application data
- Headers
- Body

##### Optional fields

With the exception of `Type` and `Status`, all of these are optional and MAY be omitted if they are empty and the underlying encoding allows this without introducing ambiguity. For example, the JSON encoding can easily handle that whereas a binary encoding may have a difficult time to omit certain fields.

##### Type

The field `type` in a request defines the semantics of the given request. Defining a particular request type usually comes with defining the headers which are usable within this request.

##### Status

The design of the `status` field is similar to the status-code in HTTP. It's goal is to be easily machine-readable and assign semantics to status codes that are related to each other. See [[1], Section 3.3]( https://tools.ietf.org/html/rfc3117#section-3.3) for more information on the design of status codes.

##### Application data

This transport protocol defines generic functionality for our application protocol `GANP`. In order to avoid conventions like prefixing custom headers with `X-` as in HTTP, the `REQUEST` and the `RESPONSE` frame reserves a dedicated place for sending application data to other nodes. 

##### Headers

`Headers` and `Body` are supposed to be used by the application protocol (for the beginning, just GANP). Similar to HTTP, application protocols MAY include some kind of 'Content-Type' in the protocol headers in order to describe the encoding of the payload.

In addition, protocol headers also encode compatibility information. Each header is available in two variants:

- MUST understand
- MAY ignore

If a node receives a protocol header in the `MUST understand` variant in a `REQUEST` and it does not understand it, it MUST reject the request with a `SE01` response. See section [Sender Error Responses](#sender-errors-se00-se99) for further details. Headers encoded as `MAY ignore` are ok to be not understood. Nodes may simply ignore them as if they were not there.

If a node receives a `RESPONSE` frame with a header that is marked as `MUST understand` but it does not understand it, it MUST stop processing this response. The reason behind this is that the responding party demanded understanding of this header in order to continue. This is most likely due to a backwards-incompatible change in a protocol where the receiving party would act in an incompatible way if it did not take the mandatory header into account.

##### Status code families

Similar to HTTP, this protocol divides status codes into different families. By separating semantically different responses from each other, programs are able understand these semantics and act upon them. The simplest example is probably successful versus non-successful responses. Even within non-successful responses, one can distinguish between errors caused by the sender and errors caused by the receiver (400 vs 500 in HTTP).

The purpose of status codes is to allow nodes to act in pre-defined ways for certain error/success scenarios. For example, a node could be configured auto-retry a request for certain errors if it knows how to resolve them.

In order to avoid confusion with HTTP, the status code families are defined as a pair of characters. Individual codes within the family have a particular meaning.

- OK00 (read as OK-00)
- SE00 (read as **S**ender**E**rror-00)
- RE00 (read as **R**eceiver**E**rror-00)

This protocol reserves the status codes 00 to 19 for all of the families. Application protocols that build on top of this transport protocol MUST NOT redefine any status codes within this range. However, they are highly encouraged to define status codes started from `20` for their own purposes.

If a node doesn't understand a particular status code, it should treat it as the generic one for the particular family. For example, if the status code `OK58` is not understood, node should continue as if the status code is `OK00`.

###### Successful responses (OK00-OK99)

All kinds of successful responses go into the `OK` family. `OK00` to `OK19` are reserved for use within this transport protocol.

1. OK00 - Successful

    A response with `OK00` indicates that the responding node understood the request and processed it successfully.

###### Sender errors (SE00-SE99)

If a node responds with a status code from the `SE` family, this means that it was unable to process this request due to its content (similar to the HTTP 400 family). Sending nodes SHOULD NOT resend such a request without modification.

Looking at these responses, it might at first be confusing why they are listed in the `SE` category even though it is usually the receiving end that is unable to process the request because it doesn't meet certain requirements. The reason why it is still a `SenderError` is because the sender can potentially adjust the request and try again if it has this kind of error handling built in. The receiving end however probably needs an upgrade to a newer version so that it understands these requests. It is therefore the sender's "fault" that the two nodes could not successfully communicate.

1. SE00 - Bad request

    A response with the status code `SE00` indicates that the request sent by sender was malformed and could not be processed. This is the most generic error that can be returned by a node. If no other errors code fits as to why the request was faulty, this one can be returned.

2. SE01 - Unsupported mandatory header

    Whenever a node receives a header that it does not understand, it should return a response with the SE01 status code. Responses with the SE01 status code MUST include the header `unsupported_headers` with its value set to a list of mandatory headers it didn't understand. If a node does not understand multiple headers, it SHOULD list all of them.

3. SE02 - Unknown request type

    A response with the status code `SE02` indicates that the sender sent a request with a `type` that was not understood.

###### Receiver errors (RE00-RE99)

Responses from the `RE` family indicate that the receiving node encountered an error during process of the request. A sending node MAY infer that the request itself was fine but the receiving end failed to process it due to an internal error which the sending end MUST NOT be held responsible for.

1. RE00 - Internal error

    `RE00` indicates that the node encountered an internal, unexpected error during processing of the request.

### Headers

A header is a key-value structure. Its actual structure is to be defined by an encoding of this protocol. This section just describes the semantics that need to be encoded in a header.

A header's key acts as its identifier. A header's value MUST encode the following information:

1. A flag to indicate the variant of this header

    Headers appear in two variants: MUST understand and MAY ignore. This allows nodes with different versions of a particular protocol to enforce rejection of a message that contains a header that the other node does not understand. It also facilitates incremental rollout of features. At first, a header can be declared as `MAY ignore`. At some point, an implementation might demand that another node understands a particular header by only sending the `MUST understand` variant.

2. A value: The actual value that is associated with the header

    A header's value MUST be a single token, e.g. no nested structure etc. This is to facilitate comparisons of header-values. If a value needs to parameterized further, `parameters` should be used. Every header MUST have a value.

3. Parameters

    Parameters are to be used to further specify a header's value. Parameters are a key-value structure with **unique** keys. Similar to the actual header value, both keys and values must be single tokens. Parameters are optional and may therefore be empty or left out if the encoding allows this.

Headers can occur multiple times, e.g. a single header-key can map to multiple values where all of them have one `value` and `parameters`. How this is represented depends on the encoding.

Splitting headers up into `value` and `parameters` was done for the following reasons:

1. Having a single `value` facilitates comparison of header values: A node can determine whether or not they understand a particular header just by looking at the `value`. If they don't understand the `value`, they also cannot understand its `parameters`.
2. It is more consistent: By having `value` and `parameters`, the general structure of a header is always the same, independent of the actual data. This allows for easier implementation in statically-typed languages.
3. Without a dedicated `parameters` field, there could be a clash with a parameter named `value`, if they would just be next to the original `value` field.

### JSON Encoding

This section defines a text-based encoding of the above concepts. Later RFCs might define a binary encoding in order to increase efficiency.

Nodes MUST use UTF-8 for the actual character encoding. (JSON technically requires that but just to be sure, it is stated here again.)

A protocol needs to somehow encode, where messages start and where they end. In the JSON encoding of BAM, this is solved through newlines. Thus, each message MUST be on a single line.

In the following sections

#### Frames

Each frame is encoded as a JSON-object with the following schema:

```json
{
  "$id": "https://comit.network/transport-protocol/frame.json",
  "type": "object",
  "definitions": {},
  "$schema": "http://json-schema.org/draft-07/schema#",
  "properties": {
    "type": {
      "$id": "/properties/type",
      "type": "string",
      "title": "The frame type",
      "default": "",
      "examples": [
        "REQUEST"
      ]
    },
    "id": {
      "$id": "/properties/id",
      "type": "integer",
      "title": "The frame id",
      "default": 0,
      "examples": [
        0
      ],
      "minimum" : 0,
      "maximum" : 4294967295
    },
    "payload": {
      "$id": "/properties/payload",
      "type": "object"
    }
  }
}
```

Example:

```json
{
    "type": "REQUEST",
    "id": 10,
    "payload": {}
}
```

The field `payload` holds the data for all the different frames and looks different for every type. Implementations SHOULD deserialize it into a generic data structure so that they can handle all types of frames.

#### Encoding of the individual frame types

##### Request

For the `REQUEST` frame, the `payload` looks like this:

```json
{
    "type": "...",
    "application_data": {},
    "headers": {},
    "body": {},
}
```

##### Response

The `payload` of a `RESPONSE` frame shares the same encoding as the `REQUEST` frame. As noted above, responses don't have a `type` but instead a mandatory `status` field.

```json
{
    "status": "...",
    "application_data": {},
    "headers": {},
    "body": {},
}
```

#### Header

This section defines how headers are encoded in the JSON-based text encoding. 

##### Representation

Let's start off with an example:

```json
"_source_ledger" : {
    "value": "Bitcoin",
    "parameters": {
        "network": "mainnet"
    }
}
```

In the above example:
- "Source-Ledger" is the header-key.
- The underscore in the beginning denotes that this header MAY be ignored if not understood. Respectively, if the key does_not start with an underscore, the header is mandatory and MUST be understood by the node.
- `value` is the header-value (see [Headers - Requirement 2](#headers))
- `parameters` encode the parameters of the header.

##### Default values

Implementations MUST NOT fail if they receive a header without a `parameters` field but rather default to an empty object (which is equivalent to no parameters).

##### Multiple values

If the header supports multiple values it MUST be represented as list (and if it is only a single value it MUST be a single object). For example, consider a header "supported_ledgers" which lists the ledgers a node supports:

```json
"supported_ledgers" : [
    {
        "value": "Bitcoin",
        "parameters": {
            "network": "mainnet"
        }
    },
    {
        "value": "Ethereum",
        "parameters": {
            "network": "mainnet"
        }
    }
]
```

Note how the inner structure of the header is list of the same ledger structure as defined above.

##### Compact representation

Sometimes, headers only carry one particular value. For example:

```json
"swap_protocol": {
    "value": "COMIT-RFC-003"
}
```

In cases like these, where there are no parameters, implementations can choose to use the compact representation which looks like this:

```json
"swap_protocol": "COMIT-RFC-003"
```

Implementations MUST be able to process compact representations. They MUST treat them identical to the version with only a `value` field.

##### Data types

In the JSON encoding, `value` can take any scalar JSON data type (except `null`), meaning:

- numbers
- booleans
- strings

The same thing applies to the values of parameters.

#### Naming conventions

- Frame types should use all caps convention. For example, `REQUEST`.
- Headers should use snake case convention. For example, `source_ledger`.
- `REQUEST` types should use all caps convention as well. For example: `SWAP`.

## References

- [1] : https://tools.ietf.org/html/rfc3117#section-3.3