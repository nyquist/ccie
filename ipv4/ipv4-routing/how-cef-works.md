# How CEF works

## Process Switching

### How it works

1. Network interface detects a new packet on the wire. The interface will receive the packet and will place it in the I/O memory. It will then send a “receive interrupt” to the processor to indicate that a new packet needs to be switched. The interrupt itself contains the header information required to switch the packet.
2. Upon getting the “receive interrupt”, the processor now runs the appropriate process to deal with this packet (based on the header information). For IPv4 it’s _ip\_input_. The _ip\_input_ process will lookup the destination in the routing table end will find the next hop address. With this address, it will perform a new lookup in the ARP cache and will find the information to rebuild the L2 header of the packet. Next, the _ip\_input_ process will rewrite the packet’s L2 header with the new information, while it’s stored in the I/O memory. The packet is then queued for delivery.
3. The processor of the outgoing queue will take the packet from the I/O memory and will send it out the wire.
4. Once transmission is done, the interface processor will let the main processor know that the packet has been sent and the memory can be freed.
5. Upon receiving the information from the outgoing interface, the processor increments its packet coutners and frees the I/O memory where the packet used to be stored.

### Load sharing

Process switching performs per-packet load sharing. This means that if there are multiple paths to reach a destination, packets will be distributed evenly between the possible paths. For unequal cost routes (EIGRP), the distribution will be inversely proportional with the metric of each path. which means more packets will be sent over a path with a lower metric.

## Fast Switching

### How it works

Fast switching has 2 “modes on operation” depending if it switched a similar packet before (to the same destination) or not. When a packet for a destination is first received, the router will use process switching with the addition that once outgoing interface and next hop are determined, they are saved in a “route cache”. So first packet is always process switched, the rest are fast switched

1. Network interface detects a new packet on the wire. The interface will receive the packet and will place it in the I/O memory. It will then send a “receive interrupt” to the processor to indicate that a new packet needs to be switched. The interrupt itself contains the header information required to switch the packet. The interrupt will also look into the route cache to determine if the information needed to switch the packet already exists or not in the route cache. If there is no entry, the packet is process switched.
   * While performing the standard process switching, the _ip\_input_ process will also update the route cache with the information needed to rebuild packets for the determined next hop. Subsequent packets with similar destination will be switched according to the route cache entry.
2. If a useful entry is found in the route cache, the interrupt software will rewrite the header and will also determine the outgoing interface.
3. The processor of the outgoing queue will take the packet from the I/O memory and will send it out the wire.
4. Once transmission is done, the interface processor will let know the main processor that the packet has been sent and the memory can be freed.
5. Upon receiving the information from the outgoing interface, the processor increments its packet counters and frees the I/O memory where the packet used to be stored.

Since the cache entries are added only for the first packet, there is a challenge to keep the cache updated with network changes. There are mechanisms that will invalidate entries based on network events (E.g when an ARP entry for a next hop changes, or when the route for a destination changes) and there is the cache aging process that will randomly invalidate a part of the cache (1/20th or 1/5th every minute).&#x20;

It’s understandable now, that in environments where the paths in the network are not stable enough, the caching mechanism will not be able to cope with the network changes, resulting in a lot of packets being process switched.

Intial versions of fast switching used hashed tables to store information, but later versions use a [radix tree](https://en.wikipedia.org/wiki/Radix\_tree). The issue with using the radix tree, is that the subnet information is not used, so routes that share the same initial bits may get overlapped entries in the radix tree. To avoid such situations, IOS imposes some rules on how the tree is built. Further developments (Optimized switching and CEF) resolved this issue by using an [mtree](https://en.wikipedia.org/wiki/M-tree) instead of the radix tree.

### Load sharing

Using fast switching, even if there are multiple paths to a destination, a router will keep using the path determined by the first processed switched packet until that path is invalidated. This can happen even randomly, so there is a lack of determination in regards to how packets will be delivered. This may generate link polarization depending on how things work out.

### **Optimized switching**

Optimized switching is an improved version of fast switching that applies to IPv4 only. The improvement is mostly related to the way the route cache is organized, as a [mtree](https://en.wikipedia.org/wiki/M-tree) with a depth of 4, where each node has 256 children. This maps to the 4 bytes in an IP address, making it easy to store information for any destination. The other rules of Fast switching still apply.

## CEF – Cisco Express Forwarding

CEF works by building 2 structures:

* **CEF table,** aka **FIB** (Forwarding Information Base) – implements the routing table in an easy-to-search [mtree](https://en.wikipedia.org/wiki/M-tree) data structure. The leaves of the tree store pointers into the adjacency table. It is stored in TCAM tables. When a topology change occours, the routing table is updated and the changes are reflected in the FIB as well.
* **Adjacency table** – holds information regarding the L2 fields that needs to be overwritten over the incoming packet’s header and it is populated automatically from the ARP table or other L2 protcols.

Since entries in the CEF table and the adjacency table are all updated when the routing tabel or the L3 to L2 information tables (ARP, Framer Relay maps) change, the CEF information is always up to date and no aging is needed (as with Fast switching).

### CEF table (FIB)

The main difference between a route table and FIB is that when FIB entries are updated, their next-hop doesn't have any additional dependencies and already resolves to a next hop that is attached. \
If route for prefix A has the next-hop B and route to B has the next-hop C (direclty connected) then the FIB entry for A will show the next-hop C.

```
R# sh ip cef
Prefix               Next Hop             Interface
0.0.0.0/0            no route
0.0.0.0/8            drop
0.0.0.0/32           receive              
10.10.10.0/30        attached             Ethernet0/0
10.10.10.0/32        receive              Ethernet0/0
10.10.10.1/32        receive              Ethernet0/0
10.10.10.2/32        attached             Ethernet0/0
10.10.10.3/32        receive              Ethernet0/0
127.0.0.0/8          drop
192.168.100.0/24     attached             Ethernet0/1
192.168.100.0/32     receive              Ethernet0/1
192.168.100.1/32     receive              Ethernet0/1
192.168.100.255/32   receive              Ethernet0/1
192.168.110.0/24     10.10.10.2           Ethernet0/0
224.0.0.0/4          drop
224.0.0.0/24         receive              
240.0.0.0/4          drop
255.255.255.255/32   receive              

```

* no route - there's no route to this destination. Should only appear for default route
* drop - traffic for these destinations should be dropped. This may show up for "special" prefixes like 0.0.0.0/8, 127.0.0.0/8, 224.0.0.0/4, 240.0.0/4
* receive - traffic for these destinations is considered local by the host and will not be forwarded, but it should be further processed by the router. For example traffic destined to local IP addresses and to their respective broadcast addresses
* attached - traffic for these destinations is directly connected so traffic should be forwarded
* IP address - traffic for these destinations should be forwareded to the indicated next-hop IP address

CEF can be disabled per interface or globaly

```
R# show ip interface INTF | i CEF
  IP CEF switching is enabled
R(config)# [no] ip cef
R(config-if)# [no] ip route-cache c
```

When looking at the CEF table you can filter the output by outgoing interface or destination prefixes (with longer-prefixes match). Another handy tool when troubleshooting is the cef exact-route command that allows you to see how a packet will be forwarded:

```
R# show ip cef INTF
R# show ip cef PREFIX MASK [longer-prefixes]
R# show ip cef exact-route SRC-IP DST-IP
```

### Adjacency table

```
R# show adjacency
```

Entries in the adjacency table usually have information about outgoing interfaces and information to rebuild the packet, but there are some exceptions, that will make the packets to be process switched:

* Punt: move to the next switching layer for processing – special packets, or unsupported features
  * packets that have IP Header options
  * packets with an expiring TTL
  * packets forwarded to a tunnel interface
  * packets with unsupported encapsulation types or that are routed to interfaces with unsupported encapsulation types
  * packets that exceed the MTU value of the outgoing interface and that need to be fragmented
* Drop: drop the packet, but the prefix is checked
* Discard: drop the packet
* Null: packets for Null0 interface. Drop them
* Glean: an ARP request needs to be sent in order to “glean” L2 information

{% hint style="info" %}
[https://www.cisco.com/c/en/us/support/docs/ip/express-forwarding-cef/17812-cef-incomp.html#types](https://www.cisco.com/c/en/us/support/docs/ip/express-forwarding-cef/17812-cef-incomp.html#types)
{% endhint %}

### Load sharing

Since the data structure for the CEF table doesn’t hold actual L2 information, but rather pointers to the adjacency table, CEF is able to point some destinations to single entries in the adjacency table, while pointing other entries to a “load-share table”. When reaching the load-share table, one entry will be selected based on the load-sharing algorithm that is used (per destination or per packet). This entry will then point to one entry in the adjacency table, which will be used to rebuild the packet.\
For per destination load balancing a hash is computed out of the source and destination IP address. This hash points to exactly one of the adjacency entries in the adjacency table, providing that the same path is used for all packets with this source/destination address pair. If per packet load balancing is used the packets are distributed round robin over the available paths, similar to how process switching worked. The default mode is per destination, but this can be changed using:

```
R(config-if)# ip load-sharing {per-packet|per-destination}
!Default: per-destination 
```

When you see traceroute packets being load-shared per-packet, it is because ICMP packets are process-switched, so they do not fall into the CEF-switched traffic.

### Centrialized vs Distributed CEF

For devices that run Centralized CEF, the FIB and the Adjacency table reside on the Route Processor (RP) and the RP performs CEF forwarding.

With Distributed CEF, the RP maintains the FIB and the Adjacency table but there is also a process(IPC - InterProcess Communication) that synchronizes the RP FIB and Adjacency table to with the line cards. This way the CEF forwarding is performed by the line cards directly.
