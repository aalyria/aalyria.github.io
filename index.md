---
title: "Overview"
layout: default
nav_order: 1
---

# Spacetime Developer Guides
Welcome to the Spacetime Developer Guides! These guides provide an overview of Spacetime’s object model and step-by-step instructions on how to interact with Spacetime’s APIs. 

## Overview
Spacetime has 2 APIs: 
- The **Northbound Interface (NBI)** allows humans or applications to define and orchestrate a network. This includes functions such as specifying the time-dynamic position and orientation of platforms and antennas, defining networking parameters on each node in the network, and creating requests for service that will be scheduled and routed through the network. 
- The **Control to Data-plane Interface (CDPI)**, or Southbound Interface, allows Spacetime to control network devices and receive updates in return. This includes functions such as steering antenna beams to establish new links and configuring RF parameters like the transmit power and channel. 

To start understanding and using the APIs, read about the object model [here](/object_model). 

## Access the APIs
Find the Spacetime APIs in this [Github repository](https://github.com/aalyria/api).

Spacetime’s APIs are built on [Protocol Buffers](https://protobuf.dev/), a language-agnostic format and toolchain for serializing messages, and [gRPC](https://grpc.io/), a performant RPC framework with many sophisticated features. 