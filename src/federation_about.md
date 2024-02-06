---
title: "About the Federation"
layout: default
permalink: "/federation-about"
nav_order: 4
---

# About the Federation
Spacetime’s [Federation API](https://gitlab.corp.aalyria.com/spacetime/minkowski/-/blob/main/api/fed/v1alpha/fed.proto) addresses the growing need to allow peer networks to request and to supply network resources and interconnections between partners’ networks. This results in far greater network resilience and cost effectiveness than can be achieved by any one network alone; the increased coordination facilitates dynamic, real-time inter-network connections, which allows operators to automatically and quickly supplement gaps in network coverage or advertise unused capacity to make full use of underutilized assets.

Spacetime, acting as an SDN controller, exposes and utilizes the Federation API between partners’ networks, providing full federation orchestration capabilities. This is particularly powerful given Spacetime’s ability to bridge multiple domains, enabling it to orchestrate meshes across different networks in land, sea, air, and space.

## The power of Federation
Imagine you are a provider that serves UTs on the ground through your network of GEO satellites and ground stations. Normally, the network is fairly stable, but there are many reasons why a connection to a UT might fail – for example, a surge in usage in one part of the network might over-task the ground stations in that area, and unfortunately you have no extra capacity to serve those users because an ongoing weather event has severely attenuated all satellite links into that region. Or, maybe there aren’t any problems with broken connections at all, but you require lower latency in some regions.

This is exactly what the federation is built for. Networks within the federation receive tailored, dynamic information about different options from a variety of providers across terrestrial, HAPS, LEO, MEO, and GEO networks, and operators can use these to temporarily replace a broken or high-latency link. These options come with details such as latency, bandwidth, availability time windows, and so on, enabling you to make the most informed decisions possible.

![Interconnection example](/assets/federation/interconnection_visual.png)

In turn, you could advertise underutilized GEO satellites or ground stations, allowing you to gain some extra benefit from the assets you already have.

Other use cases include (but are not limited to):
* Reaching the internet (or some specific set of IPs) via a partner network’s ground station
* Spectrum coordination and sharing between networks

Of course, with Spacetime as your SDN controller, all of this is automatic, subject to your given preferences and constraints.

## About the API
The following are examples of information that is passed between networks in a federation:

**Physical interconnectivity information** – _what terminals can talk to what, and when_
* Interface types and capabilities (wired interfaces to 3rd party user terminals, wireless access to SDA-interoperable OCTs, interface modes supported, etc.), including a table relating expected datarate to signal quality (CNIR)
* For wireless access across providers, parameters describing any geometric constraints (field of regard, angular velocity, and acceleration constraints, etc.)
* Time-dynamic motion of the parent platforms (e.g., satellite ephemerides)

**Node/network reachability and service level information** – _for each terminal, what can be reached via that terminal_
* A list or graph-based abstraction that conveys network reachable exit points, including network address and physical interface destinations (e.g. E-NNIs).
* Additional factors like supported traffic types/encapsulations, guaranteed MTU, etc. can be used, depending on the provider’s network
* Accompanying logical edges labeled with service level attributes (e.g., latency and/or throughput across various paths in the provider networks)
* Any time-dynamic changes to reachability characteristics and committed SLAs

**Cost of interconnection** — _optional: for each reachable endpoint, what is the implied cost of routing to that endpoint_
* In many cases, making use of a network’s terminal will reduce the throughput or resilience of that satellite constellation. 
* Cost metrics enable the provider network to indicate the impact to its own system, allowing Spacetime to intelligently choose links both to maximize service and minimize impact to the provider system.
* Note that the cost to the provider may also depend on the end-to-end flows that are added to their network, affecting both ingress and egress points and intermediate hops.



The API itself is still in the development stage, so details may change fairly significantly over the course of 2024. However, the main requirements for the API are constant, and therefore the API will always provide a way for:
* Resource and planning information to flow between network controllers
* Requests for reserving resources to flow from requestor to provider
* Request confirmations to flow from provider to requestor
* Fulfillment information to flow from requestor to provider
* Fulfillment confirmation to flow from provider to requestor

The API itself can be found [here](https://gitlab.corp.aalyria.com/spacetime/minkowski/-/blob/main/api/fed/v1alpha/fed.proto). This guide will be updated as the API evolves.
