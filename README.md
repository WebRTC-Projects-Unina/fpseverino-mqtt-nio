# WebRTC Project Unina

This project was developed for the "Web and Real Time Communication Systems" course at the University of Naples Federico II (A.Y. 2025/2026).

The goal of this project is to rewrite [MQTT NIO](https://github.com/swift-server-community/mqtt-nio), a [Swift NIO](https://github.com/apple/swift-nio) based MQTT client, to adopt modern Swift concurrency features and best practices such as `async`/`await`, actors, and with-style methods.
This rewrite will become MQTT NIO v3.
Progress and planned features for MQTT NIO v3 are tracked in [swift-server-community/mqtt-nio#176](https://github.com/swift-server-community/mqtt-nio/issues/176).

The `v2` branch contains MQTT NIO version 2, the current stable release.
The `v3` branch hosts the ongoing development of version 3.
There's [a draft PR](https://github.com/WebRTC-Projects-Unina/fpseverino-mqtt-nio/pull/1) that merges all commits from the `v3` branch into `v2` for easier review.

Development of MQTT NIO v3 takes place in a fork of the original repository at https://github.com/fpseverino/mqtt-nio.
This repository serves as a mirror to keep track of the changes made in the fork.
All pull requests in the fork have been reviewed and approved by the project's original creator and maintainer, [Adam Fowler](https://github.com/adam-fowler).

You can see MQTT NIO v3 in action in [EmmeQuTiTi](https://github.com/fpseverino/EmmeQuTiTi), a sample iOS app that demonstrates the use of MQTT NIO in an iOS environment.

What follows is the original README of MQTT NIO v2.

# MQTT NIO

[![sswg:sandbox|94x20](https://img.shields.io/badge/sswg-sandbox-lightgrey.svg)](https://github.com/swift-server/sswg/blob/master/process/incubation.md#sandbox-level)
[<img src="https://img.shields.io/badge/swift-5.7-brightgreen.svg" alt="Swift 5.7" />](https://swift.org)
[<img src="https://github.com/adam-fowler/mqtt-nio/workflows/CI/badge.svg" />](https://github.com/adam-fowler/mqtt-nio/workflows/CI/badge.svg)

A Swift NIO based MQTT v3.1.1 and v5.0 client.

MQTT (Message Queuing Telemetry Transport) is a lightweight messaging protocol that was developed by IBM and first released in 1999. It uses the pub/sub pattern and translates messages between devices, servers, and applications. It is commonly used in Internet of things (IoT) technologies.

MQTTNIO is a Swift NIO based implementation of a MQTT client. It supports
- MQTT versions 3.1.1 and 5.0.
- Unencrypted and encrypted (via TLS) connections
- WebSocket connections
- Posix sockets
- Apple's Network framework via [NIOTransportServices](https://github.com/apple/swift-nio-transport-services) (required for iOS).
- Unix domain sockets

You can find documentation for MQTTNIO
[here](https://swift-server-community.github.io/mqtt-nio/documentation/mqttnio/). There is also a sample demonstrating the use MQTTNIO in an iOS app found [here](https://github.com/adam-fowler/EmCuTeeTee)
