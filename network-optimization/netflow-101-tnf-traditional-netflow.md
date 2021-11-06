# NetFlow 101 – TNF – Traditional NetFlow

## What is a flow

A flow is defined by the combination of the following seven key fields:

* Source IP address
* Destination IP address
* Source port number
* Destination port number
* Layer 3 protocol type
* Type of service (ToS)
* Input logical interface

## Gathering information

Netflow requires CEF, so before beginning, make sure you enable CEF:

```
R(config)# ip cef
```

Netflow enables a router to gather specific statistics regarding each flow. These statistics include Byte count, Packet count, Start/End Time, QoS markings, and so on. In order to enable NetFlow to gather statistics, you will have to enable it per interface, and per direction of traffic (ingress or egress):

```
R(config-if)# ip flow {ingres|egress}
```

Older IOS versions support only ingress monitoring with the command:

```
R(config-if)# ip route-cache flow
! equivalent to ip flow ingress
```

Once NetFlow is enabled, it will start gathering information about each flow of data that is passed on the specified interfaces and in the configured directions. Netflow will gather statistics for all incoming packets (including the ones destined to the router), but will only gather statistics for the transit outgoing packets (not the ones sourced by the router).\
You can see which interfaces are configured for NetFlow with the command:

```
R# show ip flow interface
```

The router uses a flow-cache to store information about each flow. Statistics about each flow are kept in the cache until the flow expires. Once expired, the flow is exported to the flow collector in order to free the resources in the cache. The netflow process accesses the cache every second and it decides what flows are expired:

* Flows that are normally ended (by receiving a TCP FIN or RST for example)
* Flows that have not been updated in the last INACTIVE period (default 15 sec)
* Long flows that haven’t been inactive more than the INACTIVE period in the last ACTIVE period (default 30 min).
* In case the cache is almost full, it starts to consider the oldest flows as expired

When a flow is expired, it can (and usually is) exported to the

```
R(config)# ip flow-cache entries SIZE
! Default SIZE: 4096
R(config)# ip flow-cache timeoute active MIN
! Default MIN: 30
R(config)# ip flow-cache timeoute inactive SEC
! Default SEC: 15
```

## Displaying NetFlow data

All this information should be visualized in some way. The simplest way is using the CLI command:

```
R# show cache flow
```

To clear flow statistics on the router, use:

```
R# clear ip flow stats
```

## Exporting NetFlow data

It is hard to do complex analysis on this data just looking at the CLI. This is why NetFlow supports exporting the data in a format that can be used by other software, called NetFlow Collectors. When exporting, the router will send small, periodic packets with information regarding the collected flow statistics to the NetFlow Collector.\
To enable NetFlow export, use:

```
R(config)# ip flow-export destination HOST UDP-PORT
R(config)# ip flow-export version VERSION
! default: 1. Best option: 9
```

Over the time, the format of NetFlow exports has changed. Before version 9, the format of the Netflow packets was fixed, but version 9 introduced the concept of templates. Because version 9 supports many types of protocols (MPLS, IPv6, Multicast) each flow may be exported using a different template. To make things easy, the router that exports the flow data, also sends the template definition periodically. The template definitions are sent as records in the netflow packets, along with (but also without) netflow data exports.\
How often is the template information sent? Periodically, based on time passed since the last template was sent or based on the number of packets sent since the last template update (whichever comes first). To configure these values, use:

```
R(config)# ip flow-export template refresh-rate PACKETS
! Default PACKETS: 20
R(config)# ip flow-export template timeout-rate MIN
! Default MIN: 30
```

You can also configure a source IP for the NetFlow packets with:

```
R(config)# ip flow-export source INTERFACE
```

### Cache Aggregation

Cache aggregation is a method of aggregating netflow information in order to reduce the bandwidth requirements of sending detailed flow information to the NetFlow Collector. A flow is added to the aggregated cache only after it expires in the main cache!

To enable cache aggregation, use the command:

```
R(config)# ip flow-aggregation cache METHOD
```

After this, additional configurations can be done:

```
R(config-flow-cache)# export destination HOST UDP-PORT
R(config-flow-cache)# export version [8|9]
! Only versions 8 and 9 support aggregation
R(config-flow-cache)# export template {refresh-rate PACKETS| timeout-rate MIN}
! Default PACKETS: 20, Default MIN: 30
R(config-flow-cache)# cache entries SIZE
! Default 4096
R(config-flow-cache)# cache timeout {active MIN | inactive SEC}
! Default MIN: 30, Default SEC: 15
R(config-flow-cache# enabled
! MUST USE to enable the cache! Otherwise, it is not used
```

Verify if it is on by running:

```
R# show ip flow export
```

### Netflow reliable export using SCTP

When netflow data are used for accounting (and billing), you need a reliable method to export netflow data. By default Netflow data uses UDP for transport which is unreliable. SCTP stand for Stream Control Transport Protocol and is a transport layer protocol (Protocol numeber 132). SCTP uses a mechanism similar to TCP to control the flow rate (it caches flows locally until the destination aggregator has enough resources to process them) and also retransmits them until they are acknowledged.\
To enable export of Netflow using SCTP, use:

```
R(config)# ip flow-export destination DEST-IP PORT sctp
```

You will enter a subconfig mode where you can define the parametors for SCTP export:

```
R(config-flow-export-sctp)#backup destination BKP-IP BKP-PORT
! Add a backup SCTP destination in case the primary fails:
R(config-flow-export-sctp)# backup failover FAILOVER-TIME
! How much to wait before declaring the primary destination failed. Default 25 msec
R(config-flow-export-sctp)# backup restore-time RESTORE-TIME
! How much to wait before declaring the primary destination active again
R(config-flow-export-sctp)# backup mode {redundant|fail-over}
! In redundant mode it keeps a connection open with the backup all the time.
! In fail-over mode it only opens the connection when needed
R(config-flow-export-sctp)# reliability {full|none|partial buffer-limit BUFFER-SIZE}
! Default: full reliability. Partial reliability only buffers a limited amount of data. None - doesn't buffer anything.
```

## NetFlow Top Talkers

Netflow top talkers enable the router to do some a limited ammount of netflow stats locally for quick viewing.

```
R(config)# ip flow-top-talkers
! Limit the matching traffic
R(config-flow-top-talkers)# match ?
  byte-range        Match a range of bytes
  class-map         Match a class
  destination       Match destination criteria
  direction         Match direction
  flow-sampler      Match a flow sampler
  input-interface   Match input interface
  nexthop-address   Match next hop
  output-interface  Match output interface
  packet-range      Match a range of packets
  protocol          Match protocol
  source            Match source criteria
  tos               Match TOS
! Sort flows by packets or bytes
R(config-flow-top-talkers)# sort-by {bytes|packets}
! Limit number of displayed flows:
R(config-flow-top-talkers)# top NUMBER
```

Then, display top talkers statistics, using:

```
R# show ip flow top-talkers
```

## Filtering and Samping – MQC based NetFlow

You can limit the type of traffic that is sampled, by using NetFlow in a service policy. First, define the type of traffic that is interesting:

```
R(config)# class-map CLASS-MAP
R(config-cmap)# match access-group ACL
```

Then, define a flow-sampler map:

```
R(config)# flow-sampler-map NETFLOW-MAP
! Define a sampling frequency
R(config-sampler)# mode random one-out-of N
! It will only sample 1 packet every N packets
```

Then add this flow-sampler-map to the CLASS-MAP in a POLICY-MAP:

```
R(config)# policy-map POLICY-MAP
R(config-pmap)# class {CLASS-MAP|class-default}
R(config-pmap-c)# netflow-sampler NETFLOW-MAP
```

Then, apply the policy on an interface:

```
R(config-if)# service-policy {input|output} POLICY-MAP 
```
