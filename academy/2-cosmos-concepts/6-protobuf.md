---
title: "Protobuf"
order: 7
description: Work with Protocol Buffers
tags: 
  - concepts
  - cosmos-sdk
---

# Protobuf

<HighlightBox type="prerequisite">

Before diving into this section, it is recommended to read the following sections:

* [Messages](./4-messages.md)
* [Modules](./5-modules.md)

</HighlightBox>

<HighlightBox type="learning">

Protobuf is a data serialization method which developers use to describe message formats. There is a lot of internal communication within an Interchain application, and Protobuf is central to how communication is done.
<br/><br/>
You can find a code example for your checkers blockchain at the end of the section to dive further into Protobuf and message creation.

</HighlightBox>

Protocol Buffers (Protobuf) is an open-source, extensible, cross-platform, and language-agnostic method of serializing object data, primarily for network communication and storage. Libraries for multiple languages parse a common interface description language to generate source code for encoding and decoding streams of bytes representing structured data.

<HighlightBox type="info">

Originally designed and developed by Google, Protobuf has been an open-source project since 2008. It serves as a basis for Remote Procedure Call (RPC) systems.

</HighlightBox>

<HighlightBox type="info">

Google provides the [gRPC project](https://grpc.io/). This universal RPC framework supports Protobuf directly.

<!-- Please add again, after finding correct section to link to: "For more information on the process, see the section entitled `Compiler Invocation`." -->

</HighlightBox>

`.proto` files contain data structures called messages. The compiler `protoc` interprets the `.proto` file and generates source code in supported languages (C++, C#, Dart, Go, Java, and Python).

### Working with Protocol Buffers

First you must define a data structure in a `.proto` file. This is a normal text file with descriptive syntax. Data is represented as a message containing name-value pairs called fields.

Next, compile your Protobuf schema. `.protoc` generates data access classes, with accessors for each field in your preferred language according to the command-line options. Accessors include serializing, deserializing, and parsing.

## Protobuf basics for Go

The [gobs](https://golang.org/pkg/encoding/gob/) package for Go is a comprehensive package for the Go environment. However, it does not work well if you need to share information with applications written in other languages. How to contend with fields that may themselves contain information to be parsed or encoded is another challenge.

For example, a JSON or XML object may contain discrete fields that are stored in a string field. In another example, a time may be stored as two integers representing hours and minutes. Protobuf encapsulates the necessary conversions in both directions. The generated classes provide getters and setters for the fields and take care of the details for reading and writing the message as a unit.

The Protobuf format supports extending the format over time in such a way that code can still read data encoded in the old format.

Go developers access the setters and getters in the generated source code through the Go Protobuf API.

<HighlightBox type="docs">

For more on encoding in the Interchain, see the [Cosmos SDK documentation on encoding](https://docs.cosmos.network/main/core/encoding.html).
<br/><br/>
Here you can find the [Protobuf documentation overview](https://docs.cosmos.network/main/core/proto-docs.html).

</HighlightBox>

## gRPC

gRPC can use Protobuf as both its interface definition language and as its underlying message interchange format. A client can directly call a method on a server application on a different machine with gRPC as if it were a local object.

gRPC is based on the idea of defining a service and specifying the methods that can be called remotely with their parameters and return types. Keep the following in mind regarding the gRPC clients and server sides:

* **Server side:** the server implements this interface and runs a gRPC server to handle client calls.
* **Client side:** the client has a stub (referred to as just a client in some languages) that provides the same methods as the server.

gRPC clients and servers can run and talk to each other in a variety of environments, from servers inside Google to your own desktop, and can be written in any of the gRPC’s supported languages. For example, you can easily create a gRPC server in Java with clients in Go, Python, or Ruby.

The latest Google APIs will have gRPC versions of their interfaces, letting you easily build Google functionality into your applications.

## Types

The core of a Cosmos SDK application mainly consists of type definitions and constructor functions. Defined in `app.go`, the type definition of a custom application is simply a `struct` comprised of the following:

* Reference to **`BaseApp`**: a reference to the `BaseApp` defines a custom application type embedding `BaseApp` for your application. The reference to `BaseApp` allows the custom application to inherit most of `BaseApp`'s core logic, such as ABCI methods and the routing logic.
* List of **store keys**: each module in the Cosmos SDK uses a multistore to persist their part of the state. Access to such stores requires a list of keys that are declared in the type definition of the app.
* List of each module's **keepers**: a keeper is an abstract piece in each module to handle the module's interaction with stores, specify references to other modules' keepers, and implement other core functionalities of the module. For cross-module interactions to work, all modules in the Cosmos SDK need to have their keepers declared in the application's type definition and exported as interfaces to other modules, so that the keeper's methods of one module can be called and accessed in other modules when authorized.
* Reference to **codec**: defaulted to go-amino, the codec in your Cosmos SDK application can be substituted with other suitable encoding frameworks as long as they persist data stores in byte slices and are deterministic.
* Reference to the **module manager**: a reference to an object containing a list of the application modules known as the module manager.

## Code example

<ExpansionPanel title="Show me some code for my checkers blockchain">

In the previous code samples, you saw something like:

```go
type StoredGame struct {
    Creator string
    Index string // The unique id that identifies this game.
    Board string // The serialized board.
    Turn string // "black" or "red"
    Black string
    Red string
    Wager uint64
}
```

With a _helpful_ note telling you that you still need to add serialization information like:

```go
type StoredGame struct {
    Creator string `protobuf:"bytes,1,opt,name=creator,proto3" json:"creator,omitempty"`
    ...
}
```

**Advance**

This is where Protobuf simplifies your activity even more. The same `StoredGame` can be declared as:

```protobuf
message StoredGame {
  string creator = 1;
  string index = 2;
  string board = 3;
  string turn = 4;
  string black = 5;
  string red = 6;
  uint64 wager = 7;
}
```

The `= 1` parts indicate how each field is identified in the serialized output and provide backward compatibility. As your application upgrades to newer versions, make sure to not reuse numbers for new fields but to keep increasing the `= x` value to preserve backward compatibility.
<br/><br/>
When _compiling_, Protobuf will add the `protobuf:"bytes..."` elements. The messages to create a game can be declared in Protobuf similarly as:

```protobuf
message MsgCreateGame {
  string creator = 1;
  string black = 2;
  string red = 3;
  uint64 wager = 4;
}

message MsgCreateGameResponse {
  string gameIndex = 1;
}
```

**Enter Ignite CLI**

When Ignite CLI creates a message for you, it also creates the gRPC definitions and Go handling code. It is relatively easy to introduce Protobuf elements into your chain using commands like the following:

```sh
$ ignite scaffold map storedGame \
    board turn black red wager:uint \
    --module checkers \
    --no-message
$ ignite scaffold message createGame \
    black red wager:uint \
    --module checkers \
    --response gameIndex
```

</ExpansionPanel>

<HighlightBox type="tip">

If you want to dive straight into coding your chain, go to [Run Your Own Cosmos Chain](/hands-on-exercise/1-ignite-cli/index.md) for more details on using Ignite CLI.
<br/><br/>
More specifically, you can jump to:

* [Store Object - Make a Checkers Blockchain](/hands-on-exercise/1-ignite-cli/3-stored-game.md) to have Ignite CLI create your first Protobuf object.
* [Create Custom Messages](/hands-on-exercise/1-ignite-cli/4-create-message.md) to have Ignite CLI create another Protobuf object, this time for messaging. You also get a walk-through of the services created.

</HighlightBox>

<HighlightBox type="synopsis">

To summarize, this section has explored:

* How Protocol Buffers (Protobuf) are an open-source, extensible, cross-platform, and language-agnostic method of serializing object data, primarily for network communication and storage, and are central to how communication is done in Interchain applications.
* How the Google-authored Remote Procedure Call (gRPC) uses Protobuf as both its interface definition language and as its underlying message interchange format, allowing a client to directly call a method on a server application on a different machine as if it were a local object.
* How a Cosmos SDK application's core mainly consists of type definitions and constructor functions, comprising a reference to the `BaseApp`, a list of store keys, a list of each module's keepers, a reference to the codec used, and a reference to the module manager.

</HighlightBox>

<!--## Next up

Look at the above code example to see how Protobuf can facilitate your activities, or go straight to the [next section](/academy/2-cosmos-concepts/7-multistore-keepers.md) for an introduction to storage types and keepers.-->
