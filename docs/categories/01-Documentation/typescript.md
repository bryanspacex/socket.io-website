---
title: TypeScript
sidebar_position: 8
slug: /typescript/
---

Starting with v3, Socket.IO now has first class support for [TypeScript](https://www.typescriptlang.org/).

## Types for the server

First, declare some types:

```ts
interface ServerToClientEvents {
  noArg: () => void;
  basicEmit: (a: number, b: string, c: Buffer) => void;
  withAck: (d: string, callback: (e: number) => void) => void;
}

interface ClientToServerEvents {
  hello: () => void;
}

interface InterServerEvents {
  ping: () => void;
}

interface SocketData {
  name: string;
  age: number;
}
```

And use them when creating your server:

```ts
const io = new Server<
  ClientToServerEvents,
  ServerToClientEvents,
  InterServerEvents,
  SocketData
>();
```

Then, profit from the help of your IDE!

The events declared in the `ServerToClientEvents` interface are used when sending and broadcasting events:

```ts
io.on("connection", (socket) => {
  socket.emit("noArg");
  socket.emit("basicEmit", 1, "2", Buffer.from([3]));
  socket.emit("withAck", "4", (e) => {
    // e is inferred as number
  });

  // works when broadcast to all
  io.emit("noArg");

  // works when broadcasting to a room
  io.to("room1").emit("basicEmit", 1, "2", Buffer.from([3]));
});
```

The ones declared in the `ClientToServerEvents` interface are used when receiving events:

```ts
io.on("connection", (socket) => {
  socket.on("hello", () => {
    // ...
  });
});
```

The ones declared in the `InterServerEvents` interface are used for inter-server communication (added in `socket.io@4.1.0`):

```ts
io.serverSideEmit("ping");

io.on("ping", () => {
  // ...
});
```

And finally, the `SocketData` type is used to type the `socket.data` attribute (added in `socket.io@4.4.0`):

```ts
io.on("connection", (socket) => {
  socket.data.name = "john";
  socket.data.age = 42;
});
```

:::caution

These type hints do not replace proper validation/sanitization of the input. As usual, never trust user input.

:::

## Types for the client

On the client side, you can reuse the same `ServerToClientEvents` and `ClientToServerEvents` interfaces:

```ts
import { io, Socket } from "socket.io-client";

// please note that the types are reversed
const socket: Socket<ServerToClientEvents, ClientToServerEvents> = io();
```

Similarly, the events declared in the `ClientToServerEvents` interface are used when sending events:

```ts
socket.emit("hello");
```

And the ones declared in `ServerToClientEvents` are used when receiving events:

```ts
socket.on("noArg", () => {
  // ...
});

socket.on("basicEmit", (a, b, c) => {
  // a is inferred as number, b as string and c as buffer
});

socket.on("withAck", (d, callback) => {
  // d is inferred as string and callback as a function that takes a number as argument
});
```

## Custom types for each namespace

Since each [Namespace](../06-Advanced/namespaces.md) can have its own set of events, you can also provide some types for
each one of them:

```ts
import { Server } from "socket.io";

// types for the main namespace
const io = new Server<ClientToServerEvents, ServerToClientEvents, InterServerEvents, SocketData>();

// types for the namespace named "/my-namespace"
interface NamespaceSpecificClientToServerEvents {
  foo: (arg: string) => void
}

interface NamespaceSpecificServerToClientEvents {
  bar: (arg: string) => void;
}

interface NamespaceSpecificInterServerEvents {
  // ...
}

interface NamespaceSpecificSocketData {
  // ...
}

const myNamespace: Namespace<
  NamespaceSpecificClientToServerEvents,
  NamespaceSpecificServerToClientEvents,
  NamespaceSpecificInterServerEvents,
  NamespaceSpecificSocketData
  > = io.of("/my-namespace");

myNamespace.on("connection", (socket) => {
  socket.on("foo", () => {
    // ...
  });

  socket.emit("bar", "123");
});
```

And on the client side:

```ts
import { io, Socket } from "socket.io-client";

const socket: Socket<
  NamespaceSpecificServerToClientEvents,
  NamespaceSpecificClientToServerEvents
  > = io("/my-namespace");

socket.on("bar", (arg) => {
  console.log(arg); // "123"
});
```

## Emitting with a timeout

Emitting with a timeout breaks the symmetry between the sender and the receiver, because the sender has an additional error argument in case of failure.

This can be fixed with a little helper type:

```ts
type WithTimeoutAck<isSender extends boolean, args extends any[]> = isSender extends true ? [Error, ...args] : args;

interface ClientToServerEvents<isSender extends boolean = false> {
  hello: (arg: number, callback: (...args: WithTimeoutAck<isSender, [string]>) => void) => void;
}

interface ServerToClientEvents<isSender extends boolean = false> {
  // ...
}

const io = new Server<ClientToServerEvents, ServerToClientEvents<true>>(3000);

io.on("connection", (socket) => {
  socket.on("hello", (arg, cb) => {
    // arg is properly inferred as a number
    cb("456");
  });
});

const socket: Socket<ServerToClientEvents, ClientToServerEvents<true>> = ioc("http://localhost:3000");

socket.timeout(100).emit("hello", 123, (err, arg) => {
  // arg is properly inferred as a string
});
```
