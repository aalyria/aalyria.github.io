---
title: "Southbound Interface Developer Guide"
layout: default
permalink: "/southbound-interface-developer-guide"
nav_order: 3
---

# Southbound Interface Developer Guide 

Spacetime’s southbound interface is the collection of services through which
devices participating in the network communicate with Spacetime.

The southbound interface is made up of the following services:
- [`Scheduling`](https://github.com/aalyria/api/blob/main/api/scheduling/v1alpha/scheduling.proto):
  The scheduling service provides methods that an agent may use to receive
  updates to its schedule of configuration changes.
- [`Telemetry`](https://github.com/aalyria/api/blob/main/api/telemetry/telemetry.proto):
  The scheduling service provides methods that an agent may use to submit
  metrics to Spacetime.

## Scheduling

A primary function of Spacetime’s southbound interface is to allow the
controller to maintain a forward-looking schedule of configuration changes on
each agent. This includes scheduling forwarding-rule configuration and
packet-processing behavior, as well as antenna targeting, channel selection, and
beam-hopping configuration.

At a high level, an agent’s schedule may be thought of as a time-series of
configuration changes.

The agent is responsible for ensuring that all configuration changes scheduled
for instants in the past have been applied in chronological order. As time
advances past a configuration change’s scheduled time, the agent applies that
change.

Spacetime may modify an agent’s schedule for two broad reasons:

1. **To extend the schedule into the future as time passes.**

   A network operator may, for example, wish to ensure that each agent always
   has a schedule that extends at least X minutes out into the future. As time
   passes and the end of the schedule nears, the SDN controller must extend the
   agent’s schedule beyond the X minute horizon.


2. **To revise a part of the schedule.**

   Spacetime may choose to revise segments of an agent’s schedule in response to
   receiving information that was not available when the schedule was first
   generated (a new weather forecast, for example, or a notification of a
   hardware failure).

To enable this, the scheduling service exposes two methods:
* `ReceiveRequests`: this method establishes a
  [bidirectional stream](https://grpc.io/docs/what-is-grpc/core-concepts/#bidirectional-streaming-rpc)
  through which Spacetime may send scheduling requests to an agent, and the
  agent may respond.
* `Reset`: this method notifies Spacetime that an agent's schedule has been
  reset. The agent must call this upon startup and after any event that resets
  the agent's schedule.

To start receiving updates to its schedule, an agent should first call `Reset`
and then call `ReceiveRequests` with a `Hello` message:
```json
"hello": {
  "agent_id": "1c1bb454-beea-47bd-841e-0a7eb234b34a"
}
```
The `agent_id` should be populated with the ID of the network node the agent is
representing: Spacetime assumes a 1:1 relationship between network nodes and
agents.

### `CreateEntryRequest` and `DeleteEntryRequest` messages

The core of the scheduling service's functionality lies in the
`CreateEntryRequest` and `DeleteEntryRequest` messages an agent may receive from
Spacetime in response to a `ReceiveRequests` call.

A `CreateEntryRequest` message requests that the agent create a new entry in its
schedule. The request will include the schedule entry contents, made up of a
scheduled time and a configuration change to initiate at that time. The
scheduled configuration change may be one of the following:
* `SetRoute` or `DeleteRoute`, describing a change to a routing table.
* `SetModemConfiguration` or `DeleteModemConfiguration`, describing a change to
  a modem's configuration. 
* `SetSrPolicy` or `DeleteSrPolicy`, describing a change to a segment-routing
  policy.

A `DeleteEntryRequest` requests that an agent delete a particular entry
from its schedule.

A `CreateEntryRequest` or `DeleteEntryRequest` may create or delete a schedule
entry whose scheduled time is in the past. In such a case, the agent should
update its configuration to reflect what it _would have been_ had the request
been received in time. This may be thought of as “replaying” the past schedule
entries (with the referenced entry created or deleted as requested) to
understand what the current configuration should be.

### The `FinalizeRequest` message

The other message type an agent may receive in response to a `ReceiveRequests`
call is a `FinalizeRequest`. This represents a promise from Spacetime that it
will not create or delete any schedule entries scheduled for before a given
instant in time. In other words, Spacetime is “finalizing” the schedule “up to”
a specified instant in time.

While an agent is not strictly required to take any action in response to such a
request, an agent may choose to use the request to trigger garbage collection of
old schedule entries that will no longer be referred to or need to be replayed.

### Additional features of the Scheduling service

#### Sequence numbers

The scheduling service is designed to operate over channels that may re-order
messages. Because some scheduling operations are sensitive to the order in which
they are applied (a “create” operation followed by a “delete” operation may
leave the schedule in a different state than if the operations were applied in
the reverse order), each `CreateEntryRequest`, `DeleteEntryRequest` and
`FinalizeRequest` is annotated with a sequence number (abbreviated `seqno`).

A sequence number places the message in the sequence of ordered operations. The
first operation will be annotated with a sequence number of 1, and each
following message will have a sequence number one greater than that of the
previous message. An agent should wait to execute a request until all previous
requests in the sequence have been received and executed.

#### Schedule-manipulation tokens

An agent’s schedule may be impacted by events out of Spacetime or the agent's
control (a restart of the agent may wipe its schedule, for example).

In such an instance, it is important that operations intended to be executed on
the agent's schedule before the event are not executed on the impacted schedule.
To enforce this, the scheduling service utilizes “schedule-manipulation tokens.”
On startup or reset, an agent generates a unique schedule-manipulation token and
reports it to Spacetime in a call to `Reset`. All `CreateEntryRequest`,
`DeleteEntryRequest`, and `FinalizeRequest` intended to be applied to the
schedule from then on will be annotated with that token. Should the agent
receive a request bearing a different token, the agent must drop the request
and reply with a `Response` message indicating a
[FAILED_PRECONDITION](https://grpc.io/docs/guides/status-codes/).

## Telemetry

Spacetime's telemetry service allows agents to push metrics to the controller.
Metrics enable Spacetime to:
* Adjust its models to better reflect reality.
* Notify operators when a model and measurements differ.
* Allow operators to confirm when a model and measurements agree.
* Facilitate analyses of network performance derived from observed data.

The telemetry service is made up of a single method: `ExportMetrics`. An agent
may call this method to push metrics to Spacetime. Each pushed datapoint
is accompanied by a timestamp indicating when the datapoint was observed.

Supported metrics include:
* Interface metrics, including operational state (up, down, etc.) and standard
  statistics (packets received, packets transmitted, etc.).
* Modem metrics, include link data rates, Es/No, and SINR.

### Example `ExportMetrics` requests

#### Interface metrics
To push an interface's metrics to Spacetime, an agent may call `ExportMetrics`
with a request message resembling the following:
```json
"interface_metrics": {
  "interface_id": "123e4567-e89b-12d3-a456-426614174000",
  "operational_state_data_points": {
    "time": {
      "seconds": 1715299444,
      "nanos": 241575746
    },
    "value": "IF_OPER_STATUS_UP"
  },
  "standard_interface_statistics_data_points": {
    "start_time": {
      "seconds": 1697243803,
      "nanos": 604660330
    },
    "time": {
      "seconds": 1715299444,
      "nanos": 893398000
    },
    "rx_packets": 50441203,
    "tx_packets": 44465327,
    "rx_bytes": 52059174455,
    "tx_bytes": 113889322681,
    "tx_errors": 2
  }
}
```

#### Modem metrics
To push a modem's metrics to Spacetime, an agent may call `ExportMetrics` with a
request message resembling the following:
```json
"modem_metrics": {
  "modem_id": "75f0380d-17cb-40fd-924d-0b5abf11d3ca",
  "link_metrics_data_points": {
    "time": {
      "seconds": 1715299444,
      "nanos": 673155651,
    },
    "tx_modem_id": "7b2d5737-901a-414c-9179-efdbbf9a2cdb"
    "data_rate_bps": 5E7,
    "esn0_db": 13.98,
    "sinr_db": 13.15,
  },
  "link_metrics_data_points": {
    "time": {
      "seconds": 1715299444
      "nanos": 673157499
    },
    "tx_modem_id": "fe7e86a5-3e37-4364-9899-62c31bcde47f",
    "data_rate_bps": 5E7,
    "esn0_db": 13.05,
    "sinr_db": 12.97
  }
}
```