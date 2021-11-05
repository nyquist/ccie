# EIGRP 101

## Starting the routing process

```
R(config)# router eigrp AS-NUMBER
! AS Number mast match between neighbors
R(config-router)# network NETWORK-ADDR [WILDCARD]
!If no wildcard is specified, the network is considered classful
```

* EIGRP will advertise routes learned by the EIGRP process and all routes that appear directly connected on the interfaces that are matched by the network command. These include static routes that point to an interface and that are matched by a network command. These routes are considered directly connected and are redistributed as internal routes.
* secondary IP addresses are not advertised when using the network command. They can only be redistributed into EIGRP
* With Split Horizon enabled(default), it will not advertise a route back on the outgoing interface of that route.
* Routes that don’t make it into the routing table are not advertised

To see the interfaces that run EIGRP use:

```
R3#sh ip eigrp interfaces
IP-EIGRP interfaces for process 345

                        Xmit Queue   Mean   Pacing Time   Multicast    Pending
Interface        Peers  Un/Reliable  SRTT   Un/Reliable   Flow Timer   Routes
Fa0/0              1        0/0        71       0/2          384           0
Se1/0.301          0        0/0         0       0/1            0           0
Lo0                0        0/0         0       0/1            0           0
```

### Split Horizon

When Split Horizon is enabled on an interface, Update and Query packets are not sent for destinations which have this interface as outgoing. This could be a problem in Hub and Spoke Frame Relay topologies. Use this command to disable Split Horizon:

```
R(config-if)# no ip split-horizon eigrp AS-NUMBER
```

Make sure you know the difference between the RIP and the EIGRP command that disables Split Horizon:

```
! EIGRP:
R(config-if)# no ip split-horizon eigrp AS-NUMBER
! RIP:
R(config-if)# no ip split-horizon
```

Some IOS implementations disable split-horizon on interface where “encapsulation frame-relay” is configured.

### Passive interfaces

When defining a passive interface, the EIGRP process will advertise the network but will not send or accept EIGRP messages on the interface.

```
R(config-router)# passive-interface INTERFACE
```

This behavior is different than RIP’s, where a passive interface would still accept RIP advertisements.

You can also enable passive interfaces by default and then disable it on the interfaces where EIGRP should run:

```
R(config-router)# passive-interface default
R(config-router)# no passive-interface INTERFACE
```

## Neighbors

When a router running EIGRP receives valid HELLOs from another router, it adds it to the neighbor list. To see the neighbors list use:

```
R# show ip eigrp neighbors [detail]
```

A HELLO message contains the AS number and the K values of the router sending the message. In order to become neighbors, two routers must share the following values:

* K values
* AS number
* Primary subnet
* Authentication

By default, when valid HELLOs are not received for an entire HOLD-TIME period, the router considers the neighbor to be down. See [Timers ](eigrp-101.md#timers)section below.

### Static Neighbors

When a neighbor is defined, all communication with it is done using unicast packets:

```
R(config-router)# neighbor NEIGH-ADDR OUT-INTERFACE
```

Unlike RIP, this will disable multicast EIGRP on the interface, so no dynamic neighbors will be discovered.\
This config should be used on Frame Relay Hub & Spoke networks when two spokes should become neighbors.

### Authentication

EIGRP supports only MD5 authentication of EIGRP messages:

1.  Define the key chain

    ```
    (config)# key chain KEY-CHAIN
    R(config-keychain)# key KEY-NUMBER
    R(config-keychain-key)# key-string KEY-NAME
    ! Optionally, define the an accept-lifetime
    R(config-keychain-key)# accept-lifetime start-time {infinte|END-TIME|duration SEC}
    ! Optionally, define the an send-lifetime
    R(config-keychain-key)# send-lifetime start-time {infinte|END-TIME|duration SEC}
    ```
2.  Apply it on the interface

    ```
    R(config-if)# ip authentication mode eigrp AS-NUMBER md5
    ! sets the authentication to MD5
    R(config-if)# ip authentication key-chain eigrp AS-NUMBER KEY-CHAIN
    ```

When sending EIGRP messages, the router uses the lowest key number among all current valid keys. When receiving EIGRP messages, the router checks the MD5 digest using all current valid keys. Both the key ID and the Key-string must match in order to form an adjacency.

## Timers

### Hello interval

It specifies how often a router sends EIGRP HELLO packates. The default timer is:

* **5 seconds** for almost all interfaces
* **60 seconds** for Frame Relay physical interfaces or multipoint subinterfaces with a bandwidth lower than T1(1544kbps)

The default value can be changed using:

```
R(config-if)# ip hello-interval eigrp AS-NUMBER SEC
```

To verify, use:

```
R# show ip eigrp interface INTERFACE detail
IP-EIGRP interfaces for process 145

                        Xmit Queue   Mean   Pacing Time   Multicast    Pending
Interface        Peers  Un/Reliable  SRTT   Un/Reliable   Flow Timer   Routes
Fa0/0              1        0/0        48       0/2          240           0
  Hello interval is 5 sec
  Next xmit serial <none>
  Un/reliable mcasts: 0/1  Un/reliable ucasts: 2/4
  Mcast exceptions: 1  CR packets: 1  ACKs suppressed: 0
  Retransmissions sent: 1  Out-of-sequence rcvd: 0
  Authentication mode is not set
  Use multicast
```

### Hold Time

If a router does not receive Hello Messages for an entire Hold Time, the router considers the neighbor to have failed. The default timer is 3xHELLO-INTERVAL:

* **15 seconds** for almost all interfaces
* **180 seconds** for Frame Relay physical interfaces or multipoint subinterfaces with a bandwidth lower than T1(1544kbps)

The default value can be changed using:

```
R(config-if)# ip hold-time eigrp AS-NUMBER SEC
```

### Active Timer

When the router sends a Query, it will wait the time specified in the active timer for a replies from its neighbors. If no reply is received in the specified interval, the route is declared dead:

```
R(config-router)# timers active-time {ACTIVE-TIME|disabled}
! Default: 180 sec
```

See [Going Active](more-eigrp-features.md#going-active-sending-queries) for details.

## Packets

EGIRP uses IP protocol 88 (RTP=Reliable Transport Protocol). EIGRP uses both unicast and mulitcast packets. Except for HELLO and ACK packets, the other packets require ACK from the neighbors. A router would retry 16 times to send a packet before neighbor relationship is reset. All packets are sourced from the primary IP address of the interface.\
**HELLO** packets are sent every HELLO-INTERVAL as multicasts to 224.0.0.10 or as unicasts to each static neighbor. They contain the K values used by the router as well as the HOLD-TIME – how much time to wait for a HELLO, before resetting adjacency. EIGRP packets are sourced from the primary address of each interface.\
**UPDATE** packets are sent as unicast when neighbors are discovered initially and as multicasts whne updates are genertated by network changes. For each route, a packet contains the prefix, prefix length, hop count, and components of the advertised metric (Bandwidth, Load, Delay, Reliability, MTU). These packets are sent only when changes occur and only to the routers that need the update. EIGRP doesn’t use periodic updates. Update packets need to be acknowledged.\
**QUERY** packets are used when “going active” – see [Going Active](more-eigrp-features.md#going-active-sending-queries). QUERY packets are sent as multicast and need to be ACKed.\
**REPLY** packets are used to reply to QUERY packets. They are sent as unicasts and need to be ACKed.\
**ACK** packets are sent as an acknowledgement to Query and Reply messages. ACK are always unicasts, and EIGRP expectes one ACK from each neighbor.\
**GOODBYE** packets are sent when the EIGRP process is shut down or restarted to inform the neighbors.

If there are many changes in the network, EIGRP messages can overwhelm a link. Use this command to limit the bandwidth used by EIGRP Updates per interface:

```
R(config-if)#ip bandwidth-percent eigrp AS-NUMBER BW-PERCENT
!default: BW-PERCENT = 50
```

## EIGRP metric

See [here](eigrp-101.md#eigrp-metric) for information about EIGRP metric and offset lists.

## EIGRP Administrative Distance

By default, EIGRP AD is 90 for internal routes, 170 for external routes and 5 for summary routes. The default can be changed using:

```
R(config-router)# distance eigrp INTERNAL-AD EXTERNAL-AD
```

The AD can be changed per routing source and destination using:

```
R(config-router)# distance AD DESTINATION-IP WILDCARD-MASK [ACL]
```

## Load Balancing

In addition to Equal Cost Load Balancing, EIGRP can perform Unequal cost load balancing. To enable this, the variance command must be used.\
When setting variance, all routes that have a metric lower then the **FD \* VARIANCE** are added to the routing table and traffic can be load balanced between them.

```
R(config-router)# variance VAR
```

## Distribute Lists

Used to filter updates going out or coming into the EIGRP process

```
! Using ACLs
R(config-router)# distribute-list ACL {in|out} [INTERFACE]
! Using Prefix-list
R(config-rotuer)# distribute-list prefix PREFIX-LIST {in|out} [INTERFACE]
! Using gateway - filter based on the source of the update:
R(config-router)# distribute-list gateway PREFIX-LIST1 [prefix PREFIX-LIST2] {in|out} [INTERFACE]
! Using route maps:
R(config-router)# distribute-list route-map ROUTE-MAP {in|out} [INTERFACE]
```

## Summarization

### Auto Summarization

By default, EIGRP performs an auto-summarization each time it crosses a border between two different major networks. To disable this behaviour and to advertise the component routes, use:

```
R(config-router)# no auto-summary
```

The auto-summary routes appear as internal routes and the metric of the summary route is the best metric from among the summarized routes. On the router doing the summarization, a route to Null0 is added for the summarized address, so that traffic that is destined for other destinations than the component routes, but inside the same major network, is discared.

EIGRP will not auto-summarize external routes unless there is a component of the same major network that is an internal route

### Manual Summarization

Manual sumamrization can be done on an interface, with no limitation of the bit boundary, using:

```
R(config-if)# ip summary-address eigrp AS-NUMBER NETWORK-ADDR NETWORK-MASK [AD] [leak-map LEAK-MAP]
```

The router will advertise the summary address instead of any component, as long as there is one component in the routing table.

By default, manual summary routes have an AD of 5 and point to Null0. Due to the low AD value, they can override other learned routes with the same prefix (like a default route). Use the concept of a floating summary route to change the default AD when defining the manual summary. Use a value of 255 to stop the summary route from getting into the routing table

Leak maps enable sending routing information for specific routes defined by a route-map, and the summary route for all other.

## Default routes

### Redistribute Static Route to 0.0.0.0 into EIGRP

```
R(config)# ip route 0.0.0.0 0.0.0.0 NEXT-HOP
R(config)# router eigrp AS-NUMBER
R(config-router)# redistribute static
```

### Define a default network

By default 0.0.0.0/0 is considered the default route, but EIGRP can advertise another route as the default route as long as it is marked as the default-network

```
R(config)# ip default-network NETWORK-ADDR
! NETWORK-ADDR is classless
```

If we have the NETWORK-ADDR with its default class (A/B/C) in our routing table, then the next step is to advertise it into EIGRP. This can be done via redistribution (of connected/static routes) or the network command (provided the matched interfaces will bring the specific prefix into the EIGRP topology).\
Things get ugly when the default-network is not int our routing table with the default class. Let’s take an example:\
If we have a loopack address with the network 5.5.5.5/24 and we want it to become the default network, then we will have to run the following commands:

```
R(config)# ip default-network 5.0.0.0
R(config)# ip default-network 5.5.5.5
```

The second command will generate a static route in the config, that will be deleted only when the default-network is removed from the config:

```
R(config)#sh run | i ip route
ip route 5.0.0.0 255.0.0.0 5.5.5.5
```

Now we have a route pointing to the Class A network 5.0.0.0 via the next-hop 5.5.5.5. We can add the 5.0.0.0 network to EIGRP via redistribution (static) to EIGRP or add the 5.5.5.5 interface to the network list. From now on, EIGRP will advertise a route to 5.0.0.0 that is also marked as a candidate default.

### Manual Summary to 0.0.0.0/0

```
R(config-if)# ip summary-address eigrp 0.0.0.0 0.0.0.0 [AD]
```

Use a lower AD not to override other default routes learned by the downstream routers.
