# Per VC Frame Relay QoS

QoS parameters can be set on Frame Relay interfaces or subinterfaces using [MQC commands](../classification-and-marking.md). The disadvantage is that the policy applies to all the traffic of the interface, and not per virtual circuit (DLCI). For this purpose, the concept of Frame Relay map-classes was developed. A map-class contains configuration information that can be applied both per interface and per DLCI.

## Defining a Frame Relay map-class

To define a map-class, use:

```
R(config)# map-class frame-relay FR-MAP-CLASS
```

Inside the map class you can define several QoS parameters that will be applied with the class.

### Queuing – FIFO, WFQ, CQ, PQ, PIPQ

The default Queuing method is FIFO, but it can be changed using one of the following:

```
! Custom Queuing
R(config-map-class)# frame-relay custom-queue-list CQ-LIST
! Priority Queuing
R(config-map-class)# frame-relay priority-group PQ-LIST
! Weighted Fair Queueing
R(config-map-class)# frame-relay fair-queue CDT DYNAMIC-QUEUES RSVP-QUEUES MAX-BUFFER
! CDT = congestive-discard-threshold. Default: 64
! DYNAMIC-QUEUES = number-dynamic-conversation-queues. Default: 16
! RSVP-QUEUES = number-reservable-conversation-queues. Default: 0
! MAX-BUFFER = max-buffer-size-for-fair-queues. Default: 600
```

There is also the option of frame-relay PVC Interface Priority (PIPQ). This means that you assign each DLCI to a Strict Priority Queue, that works similar to PQ. The DLCIs will be served in the order that results from their assignment to one of the priority queues (high>medium>normal>low). By default, all DLCIs go to the normal priority queue. This configuration is only useful when you have different types of traffic on each DLCI. To configure PIPQ, first define the priority inside the map-class:

```
R(config-map-class)# frame-relay interface-queue priority {high|medium|normal|low}
```

Then, configure the interface for PIPQ:

```
R(config-if)# frame-relay interface-queue priority [HIGH-LEN MEDIUM-LEN NORMAL-LEN LOW-LEN]
! Enables PVC Interface Priority and optionally configures the Length of every queue.
```

When you assign the map class to the interface, or to a DLCI, they will be served based on the priority assigned to each DLCI. Map-class inheritance still works!

### Traffic Shaping and Policing

For information about CIR, Bc, Be and Tc, read about [Token Bucket](https://nyquist.eu/qos-101-policing-and-shaping/#1\_Token\_Bucket). The token bucket parameters can be configured using the following commands. Pay attention to the differences between the measurement units of MQC Traffic Shaping and FR Traffic Shaping. In MQC, Bc and Be were configured in Bytes. In FR, Bc and Be are configured in bits. Traffic Shaping uses the values for Outgoing, while policing uses the values for incoming. Also note that if shaping or policing is enabled but no map-class is set, default values for CIR, Bc, Be and Tc are used.

```
R(config-map-class)# frame-relay bc [in|out] [BC]
R(config-map-class)# frame-relay be [in|out] [BE]
! Default BC: 7000, BE:0. The values are in bits!
! if in|out is not specified, it applies to both directions
R(config-map-class)# frame-relay cir [in|out] [CIR]
! Default CIR: 56000. In bps!
R(config-map-class)# frame-relay tc TC
! Default TC: 1000. In msec! Only used when CIR=0
```

The value for Tc is only used when CIR is 0. Otherwise, Tc=Bc/CIR

Another method of specifying the values for CIR, Bc and Be is by entering both PIR and CIR. However, this will only be used to compute the values for CIR, Bc and Be that are used for traffic shaping.

```
R(config-map-class)# frame-relay traffic-rate CIR PIR
! PIR = CIR+EIR (P=Peak, C=Committed, E=Excess)
! Uses a Tc of 125 msec to compute Bc and Be
```

#### **Enabling Traffic Shaping**

The size of the shaping queue can be set using:

```
R(config-map-class)# frame-relay holdq LIMIT
! Default: FIFO: 40, CBWFQ:600
```

Just applying this Frame Relay class will not be enough to enable Traffic Shaping. Additionally, you will have to enable traffic shaping per interface with:

```
R(config-if)# frame-relay traffic-shaping
```

To verify traffic-shaping parameters used on an PVC, use:

```
R# show frame-relay pvc DLCI
```

Traffic shaping can be also be monitored using:

```
R2# sh traffic-shape

Interface   Se1/0
       Access Target    Byte   Sustain   Excess    Interval  Increment Adapt
VC     List   Rate      Limit  bits/int  bits/int  (ms)      (bytes)   Active
123           56000     875    7000      0         125       875       -
! VC = DLCI
! Target Rate = CIR [bps]
! Byte Limit = Bc+Be [Bytes]
! Sustain Bits = Bc [bits]
! Excess Bits = Be [bits]
! Interval = Tc
! Increment = Tc*CIR = Bc
```

#### **Enabling Traffic Policing**

Policing only works for incoming traffic and will use the **in** values of CIR, Bc and Be. You must enable traffic policing on the physical interface before the feature is used:

```
R(config-if)# frame-relay traffic-policing
```

#### **Adaptive Traffic Shaping**

Adaptive Traffic Shaping is a method of shaping the traffic to a rate that adapts to network conditions. When a congestion notification is received (BECN or Foresight), the CIR is decreased gradually until it reaches MINCIR value. To set the MINCIR value, use:

```
R(config-map-class)# frame-relay mincir [in|out] [MINCIR]
! Default MINCIR: CIR/2
! if in|out is not specified, it applies to both directions
```

To enable adaptive shaping in response to BECN or FORESIGHT, use:

```
R(config-map-class)#frame-relay adaptive-shaping {becn|foresight}
! becn - adapts the rate in response to BECN messages
! foresight - adapts the rate in response to FORESIGHT messages
```

Another way to enable Adaptive Shaping is to set an interface congestion threshold. Once the Queue Depth is passed that threshold, the interface is considered congested, and the class-maps will start to decrease CIR down to MINCIR. To enable this functionality, use:

```
R(config-map-class)# frame-relay adaptive-shaping interface-congestion PACKETS
! Value of the Queue depth beyond which the interface is considered congested
```

When adaptive shaping is on, you can also enable reflection of FECNs as BECNs using:

```
R(config-map-class)#frame-relay fecn-adapt
```

Also, remember to enable Frame Relay Traffic Shaping on the physical interface:

```
R(config-if)# frame-relay traffic-shaping
```

### Congestion Management – Drop DE, mark ECN

When a certain congestion threshold is reached (percentage of Queue LIMIT), you can configure the router to drop packets marked with the DE bit or to mark them with ECN bits – which translates in generating FECNs.

```
R(config-map-class)#frame-relay congestion threshold {de|ecn} THRESHOLD
! de - drop packets marked with DE. Default THRESHOLD: 100
! ecn - mark packets with ECN. Default THRESHOLD: 100
```

To configure the Queue LIMIT, use:

```
R(config-map-class)# frame-relay holdq LIMIT
```

Apart from this per-VC method, you can also configure a Congestion management mechanism that applies to the entire interface. See [Interface Based Congestion Management](./)

### Fragmentation

To enable fragmentation on a VC, use:

```
R(config-map-class)#frame-relay fragment BYTES
```

### IP RTP Priority

To enable IP RTP Priority, use:

```
R(config-map-class)# frame-relay ip rtp priority UDP-START UDP-END KBPS
```

Frame Relay Traffic Shaping (FRTS) and Frame Relay Fragmentation (FRF.12) must be configured in order for Frame Relay IP RTP Priority to work\
For details, see [IP RTP Priority](../congestion-management/ip-rtp-priority.md)

### Applying an MQC Policy

To use an MQC service policy for a specific DLCI, use it in a map-class:

```
R(config-map-class)# service-policy POLICY-MAP-NAME
```

## Applying a map-class

### Physical interface

A map-class applied on the physical interface will be applied to all PVCs of the Physical interface and any of its subinterfaces:

```
R(config-if)# frame-relay class CLASS-NAME
```

For some functions of a class map you also have to explicitly enable their usage on the physical interface:

```
! Enable Shaping
R(config-if)# frame-relay traffic-shaping
! Enable Policing:
R(config-if)# frame-relay policing
```

These commands can only be ran on physical interfaces.

### Subinterface

A map-class applied on a subinterface will be applied to all PVCs of the subinterface and will override the settings of the map-class defined on the physical interface. To configure it, use:

```
R(config-subif)# frame-relay class CLASS-NAME
```

### Per VC (DLCI)

A map-class applied on a DLCI will be applied only to that DLCI and will override the settings of the map-class defined on the physical interface or on its subinterface. To configure it, first enter the DLCI configuration mode with any of the following commands:

```
R(config-if)# frame-relay interface-dlci DLCI
R(config-if)# frame-relay map PROTOCOL ADDRESS DLCI [broadcast]
```

Then apply the map class, with:

```
R(config-fr-dlci)# class CLASS-NAME
```

## Monitoring

You can monitor the QoS settings applied on each pvc by running the command:

```
R# show frame-relay pvc [DLCI]
```

When a map-class is applied on an interface or DLCI, this command will display additional information regarding the frame-relay QoS settings that apply.
