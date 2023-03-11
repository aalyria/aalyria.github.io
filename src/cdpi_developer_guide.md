---
title: "CDPI Developer Guide"
layout: default
permalink: "/cdpi-developer-guide"
nav_order: 3
---

# Control to Data Plane Interface (CDPI) Developer Guide 

Spacetime’s CDPI endpoint provides methods through which network devices receive commands from the SDN controller and report status back in turn. The software that runs on the network element and enacts commands received from the SDN controller is called the “CDPI agent.” 

The CDPI exposes the following services:
- [`NetworkControllerStreaming`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto): A bidirectional streaming interface through which the SDN controller sends commands to CDPI agents and receives control plane state information in return. 
- [`NetworkTelemetryStreaming`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto): A bidirectional streaming interface for network elements to report telemetry data to the controller. 
- [`AttenuationEnvironment`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto): A unary interface for agents to send data from sensors to the controller, which can update Spacetime’s view of the physical world. This interface is used to augment Spacetime’s weather modeling with measured weather conditions from on-device sensors.   
- [`Cdpi`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto): This interface is still under development. Conceptually, it is similar to [`NetworkControllerStreaming`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto), but it will improve the semantic representation of the communication between the agent and the controller.

## NetworkControllerStreaming
In this interface, the SDN controller sends commands to agents through [`ControlStateChangeRequest`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto)s, which represent the creation or deletion of an update to a network element’s control plane, or a ping. Agents respond with [`ControlStateNotification`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto)s, which contain the updated control plane state, a list of statuses about pending updates, and/or a response to a ping.   

To create an update through a [`ControlStateChangeRequest`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto), the controller uses a [`ScheduledControlUpdate`](https://github.com/aalyria/api/blob/main/api/common/control.proto), which wraps the update and a `time_to_enact`, the time for the agent to begin enacting the update. The controller uses this field to synchronize updates across multiple agents and identify when there are new updates that should replace existing pending ones. 

There are 4 types of updates that Spacetime issues to agents:
- Beam updates: An update to a beam, such as an antenna or optical laser, according to the network topology that Spacetime’s solving engine has produced.
- Radio updates: A [cognitive engine](https://www.sciencedirect.com/topics/engineering/cognitive-engine) interface for issuing updates to radio-system parameters.
- Flow updates: An update to a router or switch’s configuration.
- Tunnel updates: An update to establish a tunnel with encapsulation/decapsulation rules and encryption/decryption policies.  

For each of these updates, the controller sends a message corresponding to the update (e.g. [`BeamUpdate`](https://github.com/aalyria/api/blob/main/api/common/control_beam.proto)), and the agent responds with a state message (e.g [`BeamStates`](https://github.com/aalyria/api/blob/main/api/common/control_beam.proto)) to reflect the outcome of the update. 

These state messages get sent back to the controller through  [`ControlStateNotification`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto)s. The `priority` field allows the controller to establish multiple control plane paths to reach a network element, and each is tried in ascending order of priority.

To delete an update through a [`ControlStateChangeRequest`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto), the controller sends a [`ScheduledControlDeletion`](https://github.com/aalyria/api/blob/main/api/common/control.proto), which contains the ID of the intended network element and the IDs of the updates to delete.

**Beam Update and State**  
A [`BeamUpdate`](https://github.com/aalyria/api/blob/main/api/common/control_beam.proto) allows the controller to modify parameters on an antenna or optical laser. The `radio_config` field configures radio parameters such as the transmit and receive channel. The controller can also instruct the agent to load a specific modem profile or SDR firmware image through the  `modem_config_id` field. This is useful for when firmware updates are being pushed to network elements, and the controller needs to set which version is used. The `target_interface_id` and `target_id` fields set a new pointing target, and the `acquisition_info` field contains the coordinates, ADS-B transponder and MAC address of the target. The `establishment_timeout` sets a timeout on enacting the update, beyond which a status of `DEADLINE_EXCEEDED` will be returned to the controller. 

The [`BeamStates`](https://github.com/aalyria/api/blob/main/api/common/control_beam.proto) message contains the IDs of active [`BeamTask`](https://github.com/aalyria/api/blob/main/api/common/control_beam.proto)s, which represent targeting commands issued to the network element.

**Radio Update and State**  
A [`RadioUpdate`](https://github.com/aalyria/api/blob/main/api/common/control_radio.proto) allows the controller to set radio-system parameters, such as the center frequency, channel width, and transmit power through the `tx_state` and `rx_state` fields. This update also supports implementing a TDMA (Time Division Multiple Access) schedule for multiple access radios, with a series of Unicast, Beacon, Polled, or Contention transmit slots, and Unicast or Broadcast receive slots. There is also a `modem_config_id`, like in the [`BeamUpdate`](https://github.com/aalyria/api/blob/main/api/common/control_beam.proto).  

The [`RadioStates`](https://github.com/aalyria/api/blob/main/api/common/control_radio.proto) message contains a map of each interface on a network element to its radio-system state, including the `tx_state` field, `rx_state` field, and the TDMA schedule.

**Flow Update and State**  
A [`FlowUpdate`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto) specifies the addition or deletion of a [`FlowRule`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto), which describes changes to a network element’s routing table. A [`FlowRule`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto) can match packets based on a range of IP headers (IPv4 or IPv6), generic Layer 4 headers (for TCP, UDP, SCTP, etc.), or Ethernet headers, as defined in a [`PacketClassifier`](https://github.com/aalyria/api/blob/main/api/common/network.proto) message. The `action_bucket` field describes the sequence of [`Action`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto)s to enact on the network device. Each [`Action`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto) sets a destination address through a `SetField` message, or sets the egress port for forwarding matched packets via a `Forward` message. Spacetime also supports multi-protocol label switching ([MPLS](https://www.rfc-editor.org/rfc/rfc3031)) by allowing the controller to set the MPLS label in a `SetField` message and manipulate the MPLS stack through `PushHeader` and `PopHeader` messages. Each [`Action`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto) within the [`ActionBucket`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto) is presumed to be executed sequentially on the network element. 

The [`FlowState`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto) message contains a list of active [`FlowRule`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto)s.

**Tunnel Update and State**  
A [`TunnelUpdate`](https://github.com/aalyria/api/blob/main/api/common/control_tunnel.proto) specifies the addition or deletion of a [`TunnelRule`](https://github.com/aalyria/api/blob/main/api/common/control_tunnel.proto), which describes encapsulation/decapsulation rules and encryption/decryption policies. The `EncapRule` defines the encapsulated source and destination’s IP address and port number, which form the tunnel’s outer header. Both the `EncapRule` and `DecapRule` match packets based on a [`PacketClassifier`](https://github.com/aalyria/api/blob/main/api/common/network.proto) and have ESP (Encrypted Security Payload) parameters. Spacetime’s [`EspParameters`](https://github.com/aalyria/api/blob/main/api/common/tunnel.proto) currently support defining an authentication algorithm (e.g. HMAC-SHA1-96) with an encrypted key and an encryption algorithm (e.g. AES-CBC 128) with an encrypted key. In the future, we plan to support more authentication and encryption algorithms. 

The [`TunnelStates`](https://github.com/aalyria/api/blob/main/api/common/control_tunnel.proto) message contains a list of the active [`TunnelRule`](https://github.com/aalyria/api/blob/main/api/common/control_tunnel.proto)s.

### Usage
Details regarding the implementation of this interface in CDPI agents can be found in the [`cdpi_agent/`](https://github.com/aalyria/api/tree/main/cdpi_agent) directory. To gain an understanding of these concepts, we can look at the [`extproc_agent`](https://github.com/aalyria/api/tree/main/cdpi_agent/cmd/extproc_agent) binary, which delegates the enactment of CDPI commands to a user-configured external process. 

A sample `ScheduledControlUpdate` with a `FlowUpdate` might resemble: 
```json
{
  "node_id": "my-network-node-id",
  "change": {
    "flow_update": {
      "flow_rule_id": "my-flow-rule-id",
      "rule": {
        "operation": "ADD",
        "classifier": {
          "ip_header": {
            "src_ip_range": "2607:8780:1e8:b660::/64",
            "dst_ip_range": "2604:ca00:f002:12::2/128"
          }
        },
        "action_bucket": [
          {
            "action": [
              {
                "action_type": {
                  "forward": {
                    "out_interface_id": "mmwave0"
                  }
                }
              }
            ]
          }
        ]
      }
    }
  }
}
```

Then, in our custom enactment backend, we could read the message and process the update using logic such as: 
```py
def process(req: dict[str, Any]):
    check_preconditions(req)

    flow_update = req["change"]["flow_update"]
    rule_id = flow_update["flow_rule_id"]
    rule = flow_update["rule"]
    is_add = rule["operation"] == "ADD"
    packet_classifier = rule["classifier"]
    fn = add_forwarding_rule if is_add else delete_forwarding_rule

    for bucket in rule["action_bucket"]:
        for action in bucket["action"]:
            out_iface_id = action["action_type"]["forward"]["out_interface_id"]
            fn(id=rule_id, classifier=packet_classifier, out_iface=out_iface_id)


def add_forwarding_rule(id: str, classifier: dict[str, Any], out_iface: str):
    """The logic to add a forwarding rule goes here."""
    pass


def delete_forwarding_rule(id: str, classifier: dict[str, Any], out_iface: str):
    """The logic to delete a forwarding rule goes here."""
    pass
```

## NetworkTelemetryStreaming
This service allows agents to send telemetry data to the SDN controller. The controller requests data through a [`TelemetryRequest`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto), which can optionally configure a rate for data to be published periodically. The agent responds with a [`TelemetryUpdate`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto), containing either a [`NetworkStatsReport`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto) or a [`NetworkEventReport`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto).

**NetworkStatsReport**  
This report contains maps with radio-system parameters and packet transmission data for network interfaces on the network elements, targeting and positioning data for antennas, and packet transmission data for individual [`FlowRule`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto)s. [`RadioStats`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto) store telemetry from the transmitter, such as the physical-layer data rate and transmit packet error rate, and telemetry from the receiver, such as the mean squared error of the signal received, the power of the signal at the receiver’s output and a breakdown of this power by chain on the receiver. [`InterfaceStats`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto) describe the number of packets and bytes transmitted and received, and the number of dropped packets and packet errors on the transmitter and receiver. [`BeamStats`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto) contain an antenna’s current target and its connection status (e.g seeking or locked), and information about its gimbal’s location, orientation, and pointing vector. [`FlowStats`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto) report the number of packets and bytes transmitted and received along a routing path or for a given [`FlowRule`](https://github.com/aalyria/api/blob/main/api/common/control_flow.proto). Each of these Stats messages is time-stamped. 

**NetworkEventReport**  
These reports represent time-stamped events or edge events. [`NetworkEventReport`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto)s support describing whether a link is up or down through a [`RadioEvent`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto), whether a port is up or down through a [`PortEvent`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto), and whether a network interface is enabled or disabled through an [`InterfaceEvent`](https://github.com/aalyria/api/blob/main/api/common/telemetry.proto). 

## AttenuationEnvironment
Agents implement this service to send weather data from sensors to the SDN controller through [`SensorWeatherData`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto) messages, which contain a [`WeatherDataForecast`](https://github.com/aalyria/api/blob/main/api/common/wireless_propagation.proto) and the location of the observations. This forecast stores data related to atmospheric attenuation (e.g. atmospheric pressure, temperature, water vapor pressure), rain attenuation (e.g. rain height relative to the WGS 84 ellipsoid, rain rate), and cloud and fog attenuation (e.g. cloud ceiling relative to the WGS 84 ellipsoid, cloud layer thickness, cloud liquid water density, cloud temperature). 

## CDPI
*This service is still under development.*

Conceptually, the [`CDPI`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto) service is similar to [`NetworkControllerStreaming`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto); both deliver messages to configure a network element’s control plane and receive state information in return. In the [`CDPI`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto) service, the controller sends [`CdpiResponses`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto) that contain a `request_payload`, which can be thought of as a container for [`ControlStateChangeRequest`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto)s. Unlike [`NetworkControllerStreaming`](https://github.com/aalyria/api/blob/main/api/cdpi/v1alpha/cdpi.proto) though, the agent-to-controller communication will be handled through unary interfaces that more clearly represent the outcomes of updates.  
