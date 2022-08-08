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

* CEF table, aka **FIB** (Forwarding Information Base) – implements the routing table in an easy-to-search [mtree](https://en.wikipedia.org/wiki/M-tree) data structure. The leaves of the tree store pointers into the adjacency table. It is stored in TCAM tables.
* **Adjacency table** – holds information regarding the L2 fields that needs to be overwritten over the incoming packet’s header and it is populated automatically from the ARP table, Frame Relay map table, and so on.

You can check the 2 tables with the commands:

```
R# show ip cef
R# show adjacency
```

Since entries in the CEF table and the adjacency table are all updated when the routing tabel or the L3 to L2 information tables (ARP, Framer Relay maps) change, the CEF information is always up to date and no aging is needed (as with Fast switching).

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
