# How the routing table is built

## The Routing Process

The routing mechanism involves three different processes: the routing protocols that build the routing table, the routing process that looks up the routing table and the forwarding process that inquires the routing process what to do with a packet.

Several routing protocols can run on one router. Together with the static routes, they will build the routing table. Each protocol will have its own list of destinations and how to reach them. Each protocol will try to add its own destinations to the routing table, but only some of them will make it into the routing table.

A destination is represented by a network number and a network mask. Two destinations are considered different if either the network number or the network mask are different, even if they are part of the same major network. Here’s an example: 10.0.0.0/24 is different than 10.0.1.0/24, but also 192.168.10.0/24 is different than 192.168.10.0/25.

## Building the routing table

When installing a route in the routing table, the following algorithm is used:

1. If there is just one route for a destination, that route is added to the routing table.
2. If there are several routes for a destination, the routes with the lowest administrative distance could be added to the routing table.
3. If there are several routes for a destination and they all have the same administrative distance, then the route with the best metric could be added to the routing table.
4. If there are one or more routes for a destination with equal administrative value and metric, then all these routes are added to the routing table.

| Route Source       | AD  |
| ------------------ | --- |
| Directly Connected | 0   |
| Static Route       | 1   |
| EIGRP summary      | 5   |
| BGP External       | 20  |
| EIGRP Internal     | 90  |
| IGRP               | 100 |
| OSPF               | 110 |
| IS-IS              | 115 |
| RIP                | 120 |
| EGP                | 140 |
| ODR                | 160 |
| EIGRP External     | 170 |
| BGP Internal       | 200 |
| Unknown            | 255 |

### What is the Administrative Distance?

The Administrative Distance (AD) differentiates the routes based on the routing protocol that knows about them. Usually, the default values are used, but they can be changed. The route with the lowest AD could be added to the routing table. The next tiebreaker is the metric.

### What is the metric?

Within the routes that come from the same routing protocol, the metric is the tiebreaker used to differentiate them. Since the metric is calculated differently for each routing protocol, comparing metrics for routes with different AD makes no sense.

## Looking up the best route in the routing table

### Classless routing

With classless routing, a routing table lookup always returns the longest match. This actually means any supernet that contains our destination address. If there is no destination that matches our destination IP, the default route could be used. The default route is a usually defined as 0.0.0.0/0 or 0.0.0.0 0.0.0.0 and is the lowest possible entry in the routing table that will match any route.\
Classless routing is the default behavior of the routing table, but if it was changed, you can use it again, with:

```
R(config)# ip classless
```

### Classful routing

If using classful routing, when a packet arrives, the destination address is used to route the packet.

* Is the major network of the destination address in the routing table?
  * NO: Is there a default route?
* YES: Use the default route
* NO: Drop the packet
* YES: Is there a subnet of the major network, where the destination address fits?
  * NO: Drop the packet
  * YES: Route the packet

To enable classful routing, use:

```
R(config)# no ip classless
```

## Finding the exit interface

Usually, a route for a destination points to the address of the next-hop. The router will have to make another routing table lookup to find out how to route towards the next-hop. This is done recursively until it finds a match pointing to a directly connected network, so it finds the outgoing interface.

In order to send the packet out the outgoing interface, the router must now build the Layer 2 packet. If the outgoing interface is a point-to-point interface (like a HDLC, PPP or frame-relay point-to-point subinterface), the router should have all information to send the packet to the other end.\
If the outgoing interface is a multipoint interface (ethernet, frame-relay physical interface or frame-relay multipoint subinterface) then the router must find the Layer 2 address to use in order to get to the destination.

The layer 2 address to use could be statically defined or it can be discovered using ARP on Ethernet or Inverse ARP on Frame Relay, but there can still be some issues.

Static routes can be configured to point to a next-hop address or to an outgoing interface. If the destination is not on the subnet of the outgoing interface, normal ARP will fail because no host will reply to the ARP requests. In such situations, Proxy ARP can help. A router on the subnet configured for Proxy ARP could respond with its Layer 2 address in order to receive the packet and route it towards the destination. Proxy ARP is enabled by default on Cisco Routers, but depending on the IOS version, it can be enabled or disabled per interface:

```
R(config)# R(config-if)# [no] ip proxy-arp
! The interface must be a Layer 3 interface.
```

## Load sharing

We know that a router will add into its routing table only the best route to the destination \[AD/metric]. But what if there are multiple paths that have the same \[AD/metric]. Which one should be used? It turns out the router can add multiple routes for the same destination, as long as they have the same metric. The number of routes that can exist in the routing table at one time for the same destination depends but it can be changed with:

```
R(config)# maximum-paths VAL
! Default: non-BGP: 4, BGP: 1
! Max:  Probably 16
```

The max value can be checked with:

```
R# sh ip route summary
IP routing table name is Default-IP-Routing-Table(0)
IP routing table maximum-paths is 16
```

For static routes you can always add new routes up to the maximum number of routes.

### Load sharing with EIGRP

For EIGRP you can also use the **variance** command to do [Unequal Cost Load Balancing](https://nyquist.eu/eigrp-101/#7\_Load\_Balancing). EIGRP is the only IGP that can do Unequal Cost Load Balancing because the DUAL algorithm enables it to find loop-free paths to the same destination. That’s why only routes from Feasible Successors can be used for load balancing. Also, by default, EIGRP sends traffic on each path inversely proportional to the metric. To change this, use:

```
R(config-router)# traffic-share min across interface
! Default on RIP and OSPF
```

This command will make EIGRP loadbalance only on paths that have the minimum cost. The other paths will still be in the routing table, but will not receive traffic. To revert to the default, use:

```
R(config-router)# traffic-share balanced
! Default for EIGRP. Not available for OSPF and RIP
```

### Load sharing with BGP

BGP doesn’t support load balancing by default, but it can be enabled with the same command:

```
R(config-router)# maximum-paths [ibgp] VAL
```

It will do equal cost load balancing, but it can do it proportional to the bandwidth, with the following command:

```
R(config-router-af)# bgp dmzlink-bw
```

To actually work, the bandwidth value must be set and sent between neighbors using extended communities:

```
R(config-router-af)# neighbor NEIGH-ADDR activate
R(config-router-af)# neighbor NEIGH-ADDR send-community extended
R(config-router-af)# neighbor NEIGH-ADDR dmzlink-bw
```

### Load sharing in real life

In reality, the actual paths used can differ, based on the packet switching mode that is in use. Check [this article](https://nyquist.eu/how-cef-works/#Load\_sharing) for more details
