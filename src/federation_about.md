---
title: "About the Federation"
layout: default
permalink: "/federation-about"
nav_order: 4
---

# The power of Federation

Spacetime’s [Federation API](https://github.com/aalyria/api/blob/main/api/federation/federation.proto) addresses the growing need for networks to share resources, spectrum, and to orchestrate interconnections across administrative boundaries. This results in far greater network resilience and cost effectiveness than can be achieved by any one network alone; the increased coordination facilitates dynamic, real-time inter-network connections, which allows operators to automatically and quickly supplement gaps in network coverage or advertise unused capacity to make full use of underutilized assets.

Spacetime, acting as an SDN controller, exposes and utilizes the Federation API between partners’ networks, providing full federation orchestration capabilities. This is particularly powerful given Spacetime’s ability to bridge multiple domains, enabling it to orchestrate meshes across different networks in land, sea, air, and space.

# Potential scenarios

Imagine you are an operator that serves UTs on the ground through your network of GEO satellites and ground stations. Normally, the network is fairly stable, but there are many reasons why a connection to some specific UT might fail – for example, a surge in usage in one part of the network might over-task the ground stations in that area, and unfortunately you have no extra capacity to serve those users because an ongoing weather event has severely attenuated all satellite links into that region.

Perhaps you need to re-establish connection to a specific target within a tactically responsive timeframe, and you value low latency above all else. Or perhaps you've gathered a large amount of data in an observation satellite and require a particularly high-throughput downlink to a destination IP.

This is exactly what the Federation is built for. Networks within the Federation receive tailored, dynamic information about different options from a variety of providers across terrestrial, HAPS, LEO, MEO, and GEO networks; operators can use these to temporarily replace a broken or high-latency link. The variety of options allows operators to make informed decisions and choose the services that best fit their needs.

![Interconnection example](/assets/federation/interconnection_visual.png)

Being a service provider in the Federation is just as beneficial; operators can easily advertise underutilized resources to improve their asset yield. As SDN-controlled networks begin to rise in prominence, requesters will be able to request interconnections in ever more dynamic time frames, allowing providers to make the most of any excess capacity as soon as it's available.

# About the API

Networks in a Federation operate as requester/provider peers to one another, and this API faciliates that communication.

## Interconnection points and other examples of shared data

Interconnections between two networks are bookended by an "interconnection point" from each network. Interconnection points are external representations of network interfaces or other resources that are available to requester networks for ingress or egress.

The following are examples of other information that is passed between networks in a federation:

**Physical interconnectivity information** – _what terminals can talk to what, and when_

- Interface types and capabilities (wired interfaces to 3rd party user terminals, wireless access to SDA-interoperable OCTs, interface modes supported, etc.), including a table relating expected datarate to signal quality (CNIR)
- For wireless access across providers, parameters describing any geometric constraints (field of regard, angular velocity, and acceleration constraints, etc.)
- Time-dynamic motion of the parent platforms (e.g., satellite ephemerides)

**Node/network reachability and service level information** – _for each terminal, what can be reached via that terminal_

- A list or graph-based abstraction that conveys network reachable exit points, including network address and physical interface destinations (e.g. E-NNIs).
- Additional factors like supported traffic types/encapsulations, guaranteed MTU, etc. can be used, depending on the provider’s network
- Accompanying logical edges labeled with service level attributes (e.g., latency and/or throughput across various paths in the provider networks)
- Any time-dynamic changes to reachability characteristics and committed SLAs

**Cost of interconnection** — _optional: for each reachable endpoint, what is the implied cost of routing to that endpoint_

- In many cases, making use of a network’s terminal will reduce the throughput or resilience of that satellite constellation.
- Cost metrics enable the provider network to indicate the impact to its own system, allowing Spacetime to intelligently choose links both to maximize service and minimize impact to the provider system.
- Note that the cost to the provider may also depend on the end-to-end flows that are added to their network, affecting both ingress and egress points and intermediate hops.

**To be very clear: every network is able to choose how much information they share with the rest of the Federation.**

## Scheduling a service

`ScheduleService` allows a requester to request a service from a provider.

Directly calling this RPC, without any prior knowledge about the provider network, is particularly appealing to requesters who would like the provider to do most of the computation work for selecting and maintaining the service. In order for this to work, the requester would need to include some information about their own interconnectable resources in the request.

Alternatively, the requester may choose to first gather more information about the provider's resource locations, attributes, and SLAs. Doing this can give the requester more fine-grained control over service selection, if desired. This method also enables the requester to reveal less information about themselves to the service provider. The RPCs that facilitate this are:

1. `StreamInterconnections`, which allows the requester to see a long-lived stream of the service provider's interconnection points, as well as some of their attributes.
2. `ListServiceOptions`, which allows the requester to get a list of available services from the service provider's network, along with their SLAs.

Both parties will need to share beam targeting information with one another, for the interconnection points involved in the service.

The API is still under development, and can be found [here](https://github.com/aalyria/api/blob/main/api/federation/federation.proto). This guide will be updated as the API evolves.
