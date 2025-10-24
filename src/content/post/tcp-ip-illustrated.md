---
title: "TCP/IP illustrated notes + Linux network tooling"
description: "My notes from working through the TCP/IP illustrated book and rabbit hole digging into linux tooling for networks"
publishDate: "27 Nov 2025"
updatedDate: "27 Nov 2025"
tags: ["tcp/ip", "linux", "udp", "network-tooling"]
pinned: true
---

## Jargon

- Protocol "suite" is a collection of related protocols. 
- TCP/IP is a protocol suite.
- Different protocols in a protocol suite interact with each according to the _architecture/reference model_.
- Internet Architecture is the architecture of the TCP/IP protocol suite.
- Packet switching was the main goal -> achieved via _gateways_ -> later called routers.
- TDM : time division multiplexing
- VC : virtual circuit

## Architecture constraints

- cost-effective
- many types of communication must be supported
- *Most important* It has to be able to tolerate loss of _gateways_ and _routers_.

## Design decisions

Most of the concept of "connecting" users over the protocol is based around telephones. Thus, they served as a starting point for these ideas. Problem with telephones?

- Analog ones needed a persistent and owning connection.
- Even when no one is talking the circuit is hogged by whoever holds the connection.

Solution was to break the payload into packets and switch them using gateways.
The good?
- Much better use of resources. The chunks from multiple senders can be mixed onto the network by multiplexing and then pulled apart via demultiplexing.
- Use of statistical multiplexing allows for complete utilization of our resources and it adds enough randomness to routing that the network becomes more resilient to attacks.

TBD