---
title: "Overview"
layout: default
nav_order: 1
---

# Spacetime Developer Guides
Welcome to the Spacetime Developer Guides! These guides provide an overview of Spacetime’s object model and step-by-step instructions on how to interact with Spacetime’s APIs. 

## Overview
Spacetime has 3 APIs: 
- The **Northbound Interface (NBI)** allows humans or applications to define and orchestrate a network. This includes functions such as specifying the time-dynamic position and orientation of platforms and antennas, defining networking parameters on each node in the network, and creating requests for service that will be scheduled and routed through the network. 
- The **Southbound Interface** is the collection of services through which
devices participating in the network communicate with Spacetime. This includes services through which these devices may receive schedule updates from Spacetime, and through which they may push metrics and observations to Spacetime.
- The **Federation API**, or East-West Interface, allows peer networks to request and to supply network resources and interconnections between partners’ networks. This facilitates dynamic, real-time inter-network connections, which allows operators to automatically and quickly supplement gaps in network coverage or advertise unused capacity to make full use of underutilized assets.

## Access the APIs
Find the Spacetime APIs in this [Github repository](https://github.com/aalyria/api).

Spacetime’s APIs are built on [Protocol Buffers](https://protobuf.dev/), a language-agnostic format and toolchain for serializing messages, and [gRPC](https://grpc.io/), a performant RPC framework with many sophisticated features. 