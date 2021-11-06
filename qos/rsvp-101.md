# RSVP 101

## RSVP Basics

RSVP (Resource Reservation Protocol) is an industry standard protocol used to implement IntServ QoS architecture. RSVP is used by the sender to request a certain QoS level from the routers along the path to the receiver. Devices along the way should be RSVP aware and they will let the sender know if they can implement its requirements.\
The sender first sends an RSVP request message (PATH). This message will identify the flows and what QoS level they require. The first router in the path towards destination will receive this message and if it can offer the desired QoS level it will forward the traffic to the next router towards the destination. If all routers along the path can deliver the required QoS level, the PATH message will arrive at the destination router. If it can also deliver the QoS level, it will start to send RESV messages towards the sender, letting it and all the routers along the path that the network can offer the required QoS level.\
By sending RESV messages, the receiver is the one that actually requests the QoS level, now that we know that the network can implement it. In order for the routers to maintain the reservation, senders must send PATH messages every 30 seconds.

### RSVP Sender

To configure a Cisco router as an RSVP sender, we must configure it to sende PATH messages.

```
R(config)#ip rsvp sender-host DESTINATION-IP SOURCE-IP {udp|tcp|PROTOCOL} PORT-DST PORT-SRC CIR BC
! SOURCE-IP must exist locally
! DESTINATION-IP, SOURCE-IP, PROTOCOL, PORT-DST, PORT-SRC define the flow
! CIR, BC - define the flow's requirements
```

Similarly, you can configure a router to act as a Sender Proxy and use RSVP to request QoS for a non-RSVP enabled source. This time the SOURCE-IP doesn’t have to be locally defined:

```
R(config)# ip rsvp sender DESTINATION-IP SOURCE-IP {PROTOCOL|tcp|udp} DST-PORT SRC-PORT PREVIOUS-HOP-IP PREVIOUS-INTERFACE CIR BC
```

### RSVP Receiver

To respond to PATH messages and start sending RESV messages, a receiver has to be configured accordingly:

```
R(config)# ip rsvp receiver-host DESTINATION-IP SOURCE-IP {udp|tcp|PROTOCOL} PORT-DST PORT-SRC {ff|se|wf} {load|rate} CIR BC
! DESTINATION-IP must exist locally
! ff|se| wf - Filter Type (FilterSpec)
! load|rate - Reservation Type/Service Type
```

As you can see, the source must define the Reservation Type and the Service Type that this flow will require.\
An RSVP flow is characterized by 2 data structures: FlowSpec and FilterSpec. FlowSpec contains the RSpec (Reservation Type – the class of service that the flow requires) and TSpec(Traffic Info – the Token Bucket parameters used for metering: CIR, Bc).\
FilterSpec defines the sources that can make use of the QoS reserved for the flow. There are 3 types of filters that can be used:

* **ff** – Fixed Filter – allows only 1 source, explicitly defined.
* **sf** – Shared Filter – allows multiple sources, explicitly defined.
* **wf** – Wildcard Filter – allows any source, no explicit info.

The Reservation Type can be one of the following options:

* **Guaranteed Rate Service**, which allows applications to reserve bandwidth to meet their requirements. Cisco IOS QoS uses RSVP with WFQ for this Service Type
* **Controlled Load Service**, which allows applications to have low delay and high throughput even during times of congestion. Cisco IOS QoS uses RSVP with WRED for this Service Type

As before, you can configure a router to act as a Receiver Proxy and use RSVP to respond to PATH messages in the name of a non-RSVP enabled receiver. This time the DESTINATION-IP doesn’t have to be locally defined:

```
R(config)# ip rsvp receiver DESTINATION-IP SOURCE-IP {PROTOCOL|tcp|udp} DST-PORT SRC-PORT NEXT-HOP-IP NEXT-INTERFACE {ff|se|wf} {load|rate} CIR BC
```

### Routers along the path

In order to process RSVP messages, all routers between sender and receiver must define the interfaces that are RSVP enabled and what is the maximum bandwidth reservation that RSVP flows can use. The command to enable this is:

```
R(config-if)# ip rsvp bandwidth TOTAL-RSVP-BW [SINGLE-FLOW-BW]
! RSVP-BW can also be defined as percentage of interface bandwidth
```

This command configure the Total Bandwidth that RSVP can use and the maximum bandwidth that a single RSVP flow can request\
You can verify the RSVP flows that have been installed in the QoS Database with:

```
R# show ip rsvp installed
```

## RSVP and Fair Queueing

RSVP only works with interfaces that use a form of [Fair Queueing](https://nyquist.eu/legacy-congestion-management/#22\_Weighted\_Fair\_Queuing): WFQ or CBWFQ. The RSVP flow with the highest CIR will be assigned a weight of 6, While the other flows will have a weight equal to

$$
W_{ThisFlow}= 6\frac{max(BW_{AllRSVPFlows})}{BW_{ThisFlow}}
$$

\
However, if the traffic is more than what was reserved for it, everything that is over the requirement is treated as traffic with IP\_Precedence 0.

You can enable some RSVP flows to make use of the priority queue of LLQ. This is used primarily for voice traffic, but you can also specify flow characteristics to identify the flows that can use the priority queue.

```
! Voice Like identifies most voice traffic:
R(config)# ip rsvp pq-profile voice-like 
! Or you can define your own characteristics: 
R(config)# ip rsvp pq-profile RATE [BURST [PEAK-RATE|ignore-peak-value]]
```

## RSVP DSBM

On shared media, like Ethernet, one router can act as the Designated Subnetwork Bandwidth Manager. The reason we need such a Bandwidth Manager is that not all routers will be aware of the reservations made by other routers on the same media. By proxy-ing all RSVP requests to the DSBM, it will be aware of all reservations.\
To configure a rotuer as a DSBM candidate, use:

```
R(config-if)#ip rsvp dsbm candidate [PRIORITY]
! Default PRIORITY: 64
```

The highest PRIORITY wins and in case of ties the highest IP Address wins. There is no preemption.\
The DSBM can be configured to implement a form of Admission Control by specifying the amount of traffic that can be forwarded to the shared segment without reservation:

```
R(config-if)# ip rsvp dsbm non-resv-send-limit rate  KBPS
R(config-if)#ip rsvp dsbm non-resv-send-limit burst  KBYTES
R(config-if)#ip rsvp dsbm non-resv-send-limit peak  KBPS
R(config-if)#ip rsvp dsbm non-resv-send-limit min-unit  BYTES
R(config-if)#ip rsvp dsbm non-resv-send-limit max-unit  BYTES
```

The DSBM will distribute this information to the other RSVP routers on the segment.
