# MQTT NIO v3

This project was developed for the "Web and Real Time Communication Systems" course at the University of Naples Federico II (A.Y. 2025/2026).

## Overview

The goal of this project is to rewrite [MQTT NIO](https://github.com/swift-server-community/mqtt-nio), a [Swift NIO](https://github.com/apple/swift-nio) based MQTT v3.1.1 and v5.0 client, which is part of the SSWG (Swift Server Workgroup) Sandbox Projects, to adopt modern Swift concurrency features and best practices such as `async`/`await`, actors, and with-style methods.
This re-write will become MQTT NIO v3.
Progress and planned features for MQTT NIO v3 are tracked in [swift-server-community/mqtt-nio#176](https://github.com/swift-server-community/mqtt-nio/issues/176).

### Repository Structure

The `v2` branch contains MQTT NIO version 2, the current stable release.
The `v3` branch hosts the ongoing development of version 3.
There's [a draft PR](https://github.com/WebRTC-Projects-Unina/fpseverino-mqtt-nio/pull/1) that merges all commits from the `v3` branch into `v2` for easier review of all the commits made for version 3.

Development of MQTT NIO v3 takes place in a fork of the original repository at https://github.com/fpseverino/mqtt-nio.
This repository serves as a mirror to keep track of the changes made in the fork.
All pull requests in the fork have been reviewed and approved by the project's original creator and maintainer, [Adam Fowler](https://github.com/adam-fowler).

### iOS Sample App

<p align="center"><img width="96" height="96" alt="EmmeQuTiTi" src="https://github.com/user-attachments/assets/c864261d-e53b-4608-b4b1-ea4bf6f41fc6"></p>

You can see MQTT NIO v3 in action in [EmmeQuTiTi](https://github.com/fpseverino/EmmeQuTiTi), a sample iOS app that demonstrates the use of MQTT NIO in an iOS environment for instant messaging, built as part of the same course project.

## Topics

### Channel Handlers

In Swift NIO, a `Channel` represents a connection, and every Channel has a `ChannelPipeline` that processes all data that is sent and received by the Channel.
A `ChannelPipeline` is made up of a series of `ChannelHandler`s, which are called in order, and can modify the data that is sent and received by the Channel.

MQTT NIO v2 had three separate Channel Handlers that were located in the Channel Pipeline in the following order:

- `MQTTMessageHandler`, which was responsible for encoding and decoding MQTT messages, writing packets the client wanted to send, automatically responding to incoming packets that required a response (such as `PUBREL`), and notifying the client of incoming `PUBLISH` messages.
- `PingreqHandler`, which was responsible for sending `PINGREQ` messages at regular intervals to keep the connection alive.
- `MQTTTaskHandler`, which was responsible for creating promises for operations that required a response from the broker (such as `CONNECT`), and fulfilling those promises when the corresponding response (e.g. `CONNACK`) was received. The client could then await the future associated with the promise to know when the operation was complete.

In MQTT NIO v3 the functionality of these three Channel Handlers has been merged into a single `MQTTChannelHandler`.
Having a single Channel Handler is nowadays considered a best practice in the Swift NIO community.
This change was tracked by issues [#179](https://github.com/swift-server-community/mqtt-nio/issues/179) and [#183](https://github.com/swift-server-community/mqtt-nio/issues/183), and was resolved by PRs [#1](https://github.com/fpseverino/mqtt-nio/pull/1) and [#2](https://github.com/fpseverino/mqtt-nio/pull/2).

Another common practice is including a finite-state machine in the Channel Handler, to keep track of the connection state and ensure that only valid operations are performed in each state, especially when dealing with concurrent operations.
This has been done in MQTT NIO v3 as well, tracked by [#180](https://github.com/swift-server-community/mqtt-nio/issues/180) and resolved by [#3](https://github.com/fpseverino/mqtt-nio/pull/3).

The new `MQTTChannelHandler` now handles not only Swift NIO's promises, but also Swift's native ones (`CheckedContinuation`), enabling the use of `async`/`await` in the client API.
To achieve this, a new `MQTTPromise` type has been created, that can wrap either a Swift NIO promise or a Swift continuation.

```swift
enum MQTTPromise<T: Sendable>: Sendable {
    case nio(EventLoopPromise<T>)
    case swift(CheckedContinuation<T, any Error>)
    case forget

    func succeed(_ t: T) {
        switch self {
        case .nio(let eventLoopPromise):
            eventLoopPromise.succeed(t)
        case .swift(let checkedContinuation):
            checkedContinuation.resume(returning: t)
        case .forget:
            break
        }
    }

    func fail(_ e: Error) {
        switch self {
        case .nio(let eventLoopPromise):
            eventLoopPromise.fail(e)
        case .swift(let checkedContinuation):
            checkedContinuation.resume(throwing: e)
        case .forget:
            break
        }
    }
}
```

### Client and Connection

In MQTT NIO v2, `MQTTClient` was a public class that offered the high-level client API and held a reference to a single `MQTTConnection` instance, which was an internal class responsible only for opening the TCP socket connection to the broker and setting up the Channel Pipeline.

State-of-the-art packages in the Swift NIO ecosystem, such as `PostgresNIO` and `valkey-swift`, offer both a public Connection type that represents a single connection to the server and provides all the high-level API (a complete client in itself), and a Client type that holds a pool of connections and offers load balancing and connection management features.
The Client type has all the public APIs of the Connection type (subscribe, publish, etc.) that, when called, lease a connection from the pool, perform the operation on that connection, and then release it back to the pool.

The plan is to redesign MQTT NIO v3 in a similar way.
While the `MQTTClient` and connection pool are yet to be implemented, the `MQTTConnection` type has been redesigned and is now a public `actor` type that combines the responsibilities of both the old `MQTTClient` and `MQTTConnection`.

Swift’s `actor`s are conceptually like classes that are safe to use in concurrent environments. This safety is made possible because Swift automatically ensures no two pieces of code attempt to access an actor’s data simultaneously – it is made impossible by the compiler, rather than requiring developers to write boilerplate code using systems such as locks.

The `MQTTConnection` actor has a custom executor that runs on a Swift NIO `EventLoop`, the same `EventLoop` that the underlying Channel (TCP connection) runs on.
This means we can write to the `MQTTChannelHandler` without concern for race conditions, as everything is running on the same `EventLoop`.
`MQTTConnection` now conforms to the `Sendable` protocol, making it safe to be shared across threads.
This change was tracked in [#177](https://github.com/swift-server-community/mqtt-nio/issues/177) and implemented in [#5](https://github.com/fpseverino/mqtt-nio/pull/5).

### With-Style Methods

Another best practice in the Swift ecosystem when dealing with objects that need setup and teardown is to use so-called "with-style" methods.
These methods take a closure as a parameter, create and configure the object, pass it to the closure, and then clean up the object after the closure has finished executing.
This pattern ensures that resources are properly managed and released, even in the presence of errors.

In MQTT NIO v3, you had to initialize an `MQTTClient`, call the `connect()` method to establish a connection to the broker, then when done with the client, call the `disconnect()` method to close the connection and call `shutdown()` to release all resources used by the client.
This approach was error-prone, as it was easy to forget to call `connect()` and especially `disconnect()` and `shutdown()`, meaning you could end up with an `MQTTClient` that was not connected to the broker, or with leaked resources that were not properly released.

> UX of `MQTTClient` in MQTT NIO v2
```swift
// Create the client and connect to the broker
let client = MQTTClient(
    host: "mqtt.eclipse.org",
    port: 1883,
    identifier: "My Client",
    eventLoopGroupProvider: .createNew
)
try await client.connect()

// Use the client
try await client.publish(...)

// Disconnect and shutdown the client
try await client.disconnect()
try await client.shutdown()
```

To address this, the new `MQTTConnection` actor no longer exposes the `connect()`, `disconnect()`, and `shutdown()` methods, nor does it have a public initializer.
Instead, the only way to create a `MQTTConnection` is through the `withConnection` static method that takes care of creating, configuring, connecting, disconnecting, and cleaning up the connection.
The user only needs to provide a closure that uses the connection while it is active.

> UX of `MQTTConnection` in MQTT NIO v3
```swift
// Create the connection (automatically connects to the broker)
try await MQTTConnection.withConnection(
    address: .hostname("mqtt.eclipse.org"),
    identifier: "My Client",
    logger: Logger(...)
) { connection in
    // Use the connection inside this closure
    try await connection.publish(...)
}

// After the closure finishes,
// the connection is automatically disconnected and cleaned up,
// even in case of errors
```

The code of the `withConnection` method looks something like this:

```swift
public static func withConnection<Value>(
    address: MQTTServerAddress,
    identifier: String,
    logger: Logger,
    operation: (MQTTConnection) async throws -> Value
) async throws -> Value {
    // The `connect()` method (now private) creates the connection and connects to the broker
    let connection: MQTTConnection = try await self.connect(
        address: address,
        identifier: identifier,
        logger: logger
    )

    do {
        // Perform the `operation` provided by the user
        let result = try await operation(connection)
        // The `close()` method sends DISCONNECT and cleans up the connection
        try await connection.close()
        // Returns whatever `operation` returned
        return result
    } catch {
        // In case of errors, still close and clean up the connection
        try? await connection.close()
        // Rethrow the error to the caller
        throw error
    }
}
```

This means that the user of the API no longer has to worry about connecting, disconnecting, and cleaning up the connection, as it is all handled automatically by the `withConnection` method.

This change was tracked in [#178](https://github.com/swift-server-community/mqtt-nio/issues/178) and implemented in [#5](https://github.com/fpseverino/mqtt-nio/pull/5) too.

### Subscriptions

This project also greatly overhauls the way subscriptions are managed by MQTT NIO, making it more robust and easier to use.

In MQTT NIO v2, you could subscribe to topic filters using the `subscribe()` method of `MQTTClient`.
Similarly to connecting and disconnecting, you had to remember to call `unsubscribe()` when you no longer wanted to receive messages for a topic filter.
Not doing so would result in the client continuing to receive messages from the broker for that topic filter, which could lead to unexpected behavior and resource leaks.

To actually access incoming messages, you had to create a "publish listener" (i.e. an `AsyncSequence`)

`AsyncSequence`s in Swift are types similar to traditional `Sequence`s (like arrays), but they allow for asynchronous iteration over their elements, meaning you can `await` each element as it becomes available.

"Publish listeners" in MQTT NIO v2 received all incoming `PUBLISH` messages from the broker, and you had to filter them manually to get only the messages for the topic filters you were interested in.
Moreover, the "publish listener" didn't actually return messages directly, but rather returned `Result` types that could either be a `PUBLISH` message or an error, meaning you had to handle errors manually while iterating over the messages.

> UX of subscriptions in MQTT NIO v2
```swift
// Choose a topic filter and a QoS level to subscribe to
let subscribeInfo = MQTTSubscribeInfo(topicFilter: "my-topics", qos: .atLeastOnce)

// Subscribe to the topic filter
_ = try await client.subscribe(to: [subscribeInfo])

// Create a publish listener to receive incoming PUBLISH messages
let listener = client.createPublishListener()

// Iterate asyncronously over incoming messages
for await result in listener {
    switch result {
    case .success(let publish):
        // Filter messages for the topic filter you're interested in
        if publish.topicName == "my-topics" {
            var buffer = publish.payload
            let string = buffer.readString(length: buffer.readableBytes)
            print(string)
        }
    case .failure(let error):
        print("Error while receiving PUBLISH event")
    }
}

// Unsubscribe from the topic filter when no longer needed
try await client.unsubscribe(from: ["my-topics"])
```

MQTT NIO v3 fixes a lot of paper cuts in regards to subscriptions.

First of all, just like with connections, subscriptions are now managed through a "with-style" method.
You call the `subscribe` method of `MQTTConnection`, providing a closure that receives an `AsyncSequence` (called `MQTTSubscription`) of incoming `PUBLISH` messages.
When the closure finishes executing, the corresponding `UNSUBSCRIBE` message is automatically sent to the broker, and the subscription is cleaned up.

In reality, `MQTTSubscription` is now not a simple `AsyncSequence`, but an `AsyncThrowingStream`, meaning that you iterate directly over `PUBLISH` messages, and any errors that occur while receiving messages are automatically thrown by the stream, so you don't have to handle them manually (you now iterate with the `for try await` syntax instead of the `for await` syntax used previously).

Finally, the `MQTTChannelHandler` now automatically filters incoming `PUBLISH` messages and routes them to the correct `MQTTSubscription` based on the topic filter they were subscribed to.
In MQTT v5.0, subscriptions are identified by a unique Subscription Identifier, so the `MQTTChannelHandler` uses this identifier to route messages to the correct subscription, leading to better performance compared to MQTT v3.1.1 filtering.

> UX of subscriptions in MQTT NIO v3
```swift
// Choose a topic filter and a QoS level to subscribe to
let subscribeInfo = MQTTSubscribeInfo(topicFilter: "my-topics", qos: .atLeastOnce)

// Subscribe to the topic filter
try await connection.subscribe(to: [subscribeInfo]) { subscription in
    // Iterate asyncronously over incoming messages and throw errors automatically
    for try await message in subscription {
        var buffer = message.payload
        let string = buffer.readString(length: buffer.readableBytes)
        // No need to filter messages, as only messages for "my-topics" are received here
        print(string)
    }
}

// After the closure finishes,
// the UNSUBSCRIBE message is automatically sent,
// even in case of errors
```

The subscription management redesign was tracked in [#181](https://github.com/swift-server-community/mqtt-nio/issues/181), and resolved by PRs [#11](https://github.com/fpseverino/mqtt-nio/pull/11), [#12](https://github.com/fpseverino/mqtt-nio/pull/12) and [#15](https://github.com/fpseverino/mqtt-nio/pull/15).
