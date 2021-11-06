# Frame Relay QoS

## Per VC Frame Relay QoS

See [Per VC Frame Relay QoS](./#per-vc-frame-relay-qos)

## MQC For Frame Relay

MQC can be used to apply QoS settings per interface or per VC if used in conjunction with a map-class. Special MQC configuration is available for Frame Relay interfaces in regard to [Class Based Traffic Shaping](../policing-and-shaping/). Other than that, most features works as expected with MQC.

## Per Interface Frame Relay QoS

### Per Interface Congestion Management

#### **Congestion Management**

Apart from the [Per-VC Congestion Management”](per-vc-frame-relay-qos.md), you can also enable Congestion Management on the interface:\
The parameters are the same, only the configuration mode differs:

```
R(config-if)# frame-relay congestion-management
R(config-fr-congest)# threshold {de|ecn} PACKETS
```

#### **Discard Eligible Lists**

The DE bit is used to mark traffic that could be dropped in case of congestion. You can mark traffic with the DE bit following these steps:\
First define the Discard Eligible Traffic:

```
R(config-if)# frame-relay de-list DE-LIST {protocol PROTOCOL OPTIONS]| interface INTERFACE}
! OPTIONS are similar to PQ and CQ
```

Then apply the DE-GROUP on the DLCI with:

```
R(config-if)# frame-relay de-group DE-LIST DLCI
```

This feature only supports process-switched packets and doesn’t work with fast switching or CEF, like MQC does.

### Broadcast Queue

A special queue can be defined just for broadcast traffic. It has priority over other queues when broadcasting below the max-threshold but it is policed to the max-threshold value. To define it, use:

```
R(config-if)# frame-relay broadcast-queue LIMIT BYTE-RATE PACKET-RATE
```

The max-threshold is the first rate that is violated (BYTE-RATE or PACKET-RATE)

### DLCI Priority Levels

Works on multipoint interfaces with multiple DLCIs if the Source and Destination for the DLCIs is the same. Since the same destinations is reached via different DLCIs, you have to assign traffic to each DLCI based on the Priority that it has been assigned in a Priority List. The DLCI that is used will be decided by the priority-dlci-group, not by the mapping, since you can’t map 1 destination to 2 different maps. Works on back-to-back FR connections.

```
R(config-if)# frame-relay priority-dlci-group PQ-LIST HIGH-DLCI MEDIUM-DLCI [NORMAL-DLCI [LOW-DLCI]]
! The last DLCI will be used for all the missing priorities
```

To see how traffic is assigned to each PQ-LIST, see [PQ](https://nyquist.eu/legacy-congestion-management/#24\_Priorty\_Queueing\_PQ)

### Compression

See [Header and Payload Compression](../compression-and-lfi/header-and-payload-compression.md)

### Interface based fragmentation

To configure FRF.12 end-to-end fragmentation on a Frame Relay interface, use the following command:

```
R(config-if)# frame-relay fragment SIZE end-to-end
```

You should use a LLQ method of queuing for important traffic in order to take advantage of fragmentation

### Frame Relay Voice Adaptive Traffic Shaping and Fragmentation

This feature allows a router to use fragmentation in order to allow delay sensitive voice traffic to be interleaved between other fragments, and also to adapt traffic rates if it has to send traffic in the priority queue.\
To enable this feature, first you will have to define a hierarchical MQC policy

```
!Configure a CBWFQ policy that uses LLQ for voice traffic:
R(config)# policy-map LLQ
R(config-pmap)# class VOICE
R(config-pmap-c)# priority KBPS
!Configure a shaping policy that calls the CBWFQ policy
R(config)# policy-map SHAPE
R(config-pmap)# class class-default
R(config-pmap)# shape KBPS
R(config-pmap)# policy-map LLQ
!Define a FR map-class that enables Voice Adaptive Traffica Shaping
R(config)# map-class frame-relay VATS
R(config-map-class)# service-policy output SHAPE
! Apply the service-policy on the DLCI and enable voice-adaptive fragmentation 
R(config-if)# frame-relay fragmentation voice-adaptive [deactivation SEC]
R(config-if)# frame-relay interface-dlci DLCI
R(config-fr-dlci)# class VATS
```

There is one more thing to do, enabling fragmentation. This can be done on the interface:

```
R(config-if)# frame-relay fragment SIZE
```

Or on the map-class

```
R(config-map-class)# frame-relay fragment SIZE
```

## Flavors of Frame Relay Traffic Shaping

This section is just a summary of [Petr Lapukhov’s article on this subject](http://blog.ine.com/2010/06/14/the-four-flavors-of-frame-relay-traffic-shaping/).

### Generic Traffic Shaping

You can use [Generic Traffic Shaping](../policing-and-shaping/#generic-traffic-shaping) on Frame Relay interfaces or subinterfaces. However, you can’t use it on individual DLCI’s assigned to a multipoint (sub)interface. It supports Adaptive Traffic Shaping.

### Legacy Frame Relay Traffic Shaping

[Legacy Frame Relay Traffic Shaping](./#legacy-frame-relay-traffic-shaping) uses a **map-class** to define traffic shaping parameters that can be applied per interface or per individual DLCI. It requires the use of the **frame-relay traffic-shaping** on the physical interface. With this command, all DLCIs will be shaped, using configured or default values (Default CIR=56kbps). It supports PQ, CQ, WFQ or even CBWFQ if the map-class uses a CBWFQ service policy. You can also enable fragmentation on each PVC if you configure it in the **map-class**

### MQC Frame Relay Traffic Shaping

[MQC Frame Relay Traffic Shaping](per-vc-frame-relay-qos.md) requires the use of a shaping policy-map configured with MQC that is used as the service-policy in a frame-relay map-class that is applied on an interface, subinterface or DLCI. You can only enable fragmentation at the interface level, but you can use adaptive traffic shaping commands in the MQC policy. You should not use the **frame-relay traffic-shaping** command.

### Class Based Generic Traffic Shaping

[Class Based Generic Traffic Shaping](../policing-and-shaping/#class-based-traffic-shaping) uses MQC policies that are applied on the interface or subinterface. You can do per-VC configuration if you define the classes in the class-map with:

```
R(config)#class-map CLASS-NAME
R(config-cmap)# match fr-dlci DLCI
```

Unmatched DLCIs will not be traffic-shaped, unlike with Legacy Frame Relay Traffic Shaping.
