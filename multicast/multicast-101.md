# Multicast 101

## Multicast Addressing

### Layer 3 Addressing

Multicast IP packets are sent from the source to a destination IP in the range 224.0.0.0/4 (224.0.0.0 – 239.255.255.255). It means every multicast IP address starts with 1110 bits. This range is further subdivided in several blocks. The most important ones are:

* **Local Network Control Block (aka Link Local) – 224.0.0.0/24** (224.0.0.0 – 224.0.0.255). Packets with an address in this range are local in scope are not forwarded by routers. They are usually sent out with a TTL of 1.
* **Internetwork Control Block – 224.0.1.0/24** (224.0.1.0 – 224.0.1.255)
* **Source Specific Multicast – 232.0.0.0/8** (232.0.0.0 – 232.255.255.255)
* **GLOP – 233.A.S.0/24** (233.A.S.0 – 233.A.S.255), where A and S are each 8 bits of the 16 bits AS Number. These addresses are used by companies who have  already been allocated an AS Number and can use it for multicasting over Internet
* **Administratively Scoped Addresses – 239.0.0.0/8** (239.0.0.0 – 239.255.255.255) – These addresses are considered local to the site and should be considered private traffic (similar to RFC1918 IP Addresses)

For a complete list, see [IANA](https://www.iana.org/assignments/multicast-addresses/multicast-addresses.xml)

### Layer 2 Addressing

#### **Ethernet**

Of course, each layer 3 packet is encapsulated in a Layer 2 frame. On Ethernet, the frame has the MAC address of the sender as the source and a special address derived from the Layer 3 destination IP as the destination.

The Layer 2 Multicast Addresses has an OUI of **01-00-5E, followed by a 0(zero)  and 23 more bits**. The last 23 bits will be the same as the the last 23 bits in the Layer 3 Multicast Address. But the multicast address is always in the format 1110 + 28 more bits. That means that 5 bits are lost in the process of conversion. This also means that 2^5=32 Layer 3 addresses have the same Layer 2 address. They are:

```
224.X    .Y.Z
224.X+128.Y.Z
225.X    .Y.Z
225.X+128.Y.Z
...
239.X    .Y.Z
239.X+128.Y.Z
```

#### **Frame Relay**

On Frame Relay networks several issues might appear when routing multicast traffic. First of all, you need to set the **broadcast** keyword on the frame-relay maps where you want to send multicast frames. Most of the problems can be solved using point-to-point subinterfaces, but if this is not the case, then other issues may arise when using hub-and-spoke topology with multipoint interfaces.

*   When sending multicast traffic from one spoke to another, the same interface on the hub should be both the incoming interface and part of the OIL, which is impossible. The solution is to set the pim nbma-mode, where the JOINs are tracked per DLCI, and not per interface. This will allow traffic to enter one DLCI and exit another DLCI on the same interface. Since the PIM JOINs are tracked, only PIM-SM is supported for this configuration:

    ```
    R(config-if)# ip pim nbma-mode
    ```
* Also, since the spokes will not see each other, there is no way for one spoke to do a PRUNE OVERRIDE if it needs traffic. This means that the multipoint interface will be pruned and multicast traffic will not reach the spoke that needs it

## Protocols used

Multicast uses 2 types of protocols. First, there are the protocols used by the receivers to signal to the routers that they want to receive traffic for a certain group – [IGMP](https://nyquist.eu/igmp-101/). Then, there is the protocol used by the routers to build distribution trees that connect the sources to the receivers – PIM, DVMRP (Distance Vector Multicast Protocol) or MSDP (Multicast Source Discovery Protocol) belong to this category.

## Multicast Routing Table

### Reverse Path Forwarding

Reverse Path Forwarding is the primary loop prevention mechanism used in multicast routing. It is a simple mechanism but it successfully accomplishes the task of not duplicating packets in a multipath network.

Unicast routing is normally done by forwarding the packet based on its Layer 3 destination address. In contrast, Multicast routing forwards the packet based on its Layer 3 source address, by forwarding the packet away from the source.

Reverse Path Forwarding kicks in when a multicast packet is received. The router will run an RPF check on each packet to determine if the interface it came in was the interface that points to the source of the traffic. If it does, then it forwards the packet out all\* other multicast interfaces, except the incoming interface. If the RPF check fails, the router just drops the packet. It means it didn’t come from the source, and by forwarding it, we would create a loop.\
You can see how a RPF check is resolved issuing the command:

```
R# show ip rpf IP-ADDRESS
! if it says failed, traffic is not forwarded
```

Additional RPF settings can be configured with commands that start with:

```
R(config)# ip multicast rpf ...
```

### mRoute Table

A router can only have 1 incoming interface for any entry in the mroute table. If the multicast traffic comes from multiple interfaces, the route with the best cost is chosen \[AD/metric]. In case of equal cost load balancing, the entry with the highest next-hop IP is used as incoming interface. All other interfaces will fail the RPF check.\
Different multicast protocols use different methods to perform this RPF check. PIM (Protocol Independent Multicast) uses the unicast routing table to determine the interface that points to the source. Other protocols can maintain a separate Multicast Routing table.

The multicast routing table consists of different (\*,G) and (S,G) entries. Each entry has an incoming interface and an OIL (Outgoing Interface List). The Incoming interface cannot be at the same time in the OIL, too. If a packet passes the RPF check it is sent out all interfaces in the OIL.

To enable multicast routing on a Cisco router, use:

```
R(config)# ip multicast-routing
```

To see the mroute table, use:

```
R# show ip mroute
```

The entries in the routing table have a list of flags associated to them. This list

```
D - Dense = group works in dense mode
S - Sparse - group works in sparse mode
C - Connected  = there is a directly connected receiver
L - Local = the router itself is a receiver
P - Pruned = a PRUNE was sent upstream (no neighbors or receivers downstream)
T - Shortest Path Tree = the group uses a SPT. 
        PIM-DM - Always on
        PIM-SM - Only on (S,G) entries
J - JOIN = 
        PIM-DM - can be ignored
        PIM-SM - (*,G) signals a SPT switchover. The next packet will be routed using the (S,G). Recalculate threshod once a second
               - (S,G) signals a SPT switchover. Recalculates threshold once a minute.
F - REGISTER = there's a source on this interface
R - information in the (S,G) applies to the shared tree
```

### Static mRouting

When a RPF check fails, the incoming packet is dropped. The RPF check is done by finding the outgoing interface of the best route towards the source in the unicast routing table. When this is not the intended interface we have 2 options:

1. Manipulating the unicast table – usually not a good idea
2.  Manually setting the incoming interface:

    ```
    R(config)# ip mroute SOURCE MASK { PREVIOUS-HOP | INCOMING-INTERFACE} [AD]
    ```

    You can configure floating static mroutes if you specify an AD higher than the route in the routing table.

    This entry will be used in the mroute table. It is not an actual static route, but rather a static change of the RFP result.

## Multicast Filtering

### Multicast Boundary

Multicast traffic can be filtered on an interface using:

```
R(config-if)# ip multicast boundary ACL [filter-autorp] {in|out}
! ACL - permitted groups are allowed
! filter-autorp - also filter auto-rp groups (224.0.1.39, 224.0.1.40)
```

Both data and control plane traffic are filtered with this command and it works in both directions.

### TTL Scoping

A less flexible option for filtering multicast packets is the TTL scoping feature. In this setup, you can configure a TTL value on the interface beyond which any multicast packet that reaches that interface is dropped if the TTL value of the packet is less than configure on the interface. Normally packets are dropped when the TTL value reaches 0, but this way you can drop packets with a higher TTL as well. This only applies to multicast packets, but it applies to all of them. To enable TTL scoping on an interface, use:

```
R(config-if)#ip multicast ttl-threshold [TTL-VALUE]
! Default:0
```

## Multicast Rate Limiting

Multicast traffic can be rate-limited on each interface:

```
R(config-if)# ip multicast rate-limit {in|out}  [video|whiteboard]  [group-list ACL-GRP] [ source-list ACL-SRC] KBPS
! video, whiteboard - limits traffic only for known UDP ports
! without group-list and source-list: all multicast traffic si rate-limited
! group-list - only limits traffic for groups permitted by ACL
! source-list - only limits traffic from sources permitted by ACL
```

## Mapping multicast and broadcast

Some applications only support sending or receiving broadcast traffic. Whatever you do, broadcast traffic will not be forwarded by a router. What you can do is to convert the broadcast traffic into unicast or multicast.

### Convert broadcast to unicast – Helper Address

To convert it to unicast, use the following command on the incoming interface:

```
R(config-if)#ip helper-address DESTINATION-ADDR
```

Broadcast traffic will be forwarded as unicast to the DESTINATION-ADDR.

### Forwarding UDP Broadcasts

But what traffic? By default, a router will forward only broadcasts for these UDP ports:

* TFTP – UDP 69
* DNS – UDP 53
* Time service – UDP 37 – This is not NTP
* NetBIOS Name Server – UDP 137
* NetBIOS Datagram Server – UDP 138
* BOOTP – UDP 67,68
* TACACS – UDP 49
* IEN-116 Name Service – UDP 42

You can add additional services to this list with the global command:

```
R(config)# ip forward-protocol udp PORT
! Additional protocols besides UDP can also be forwarded
```

### Convert broadcast to multicast – Helper Map

To convert broadcast to multicast and the other way around, use the **multicast helper-map** command.\
On the router that converts Broadcat to Multicast:

```
R(config-if)# ip multicast helper-map broadcast GROUP-ADDR EXTENDED-ACL [ttl TTL]
! Use EXTENDED-ACL to limit the traffic type to certain UDP Ports
```

On the router that converts Multicast to Broadcast:

```
R(config-if)# ip multicast helper-map GROUP-ADDR BROADCAST-ADDR EXTENDED-ACL [ttl TTL]
! Use EXTENDED-ACL to match only specific UDP traffic
```

### Directed Broadcast

Directed broadcasts are IP packets that are destined to the broadcast address of a remote subnet. They have the host part of the destination set to all 1s and they are routed just like unicast packets until they reach the router that has that subnet directly connected. By default, this router will drop the packet and will not forward it on the subnet. You can forward the packets as 255.255.255.255 broadcasts if you enable the following command on the interface connected to the end subnet:

```
R(config)# ip directed-broadcast [ACL]
! If ACL is used, unmatched traffic is dropped
```

### Broadcast Address

Normally, the broadcast address used by a router is 255.255.255.255. Older implementations use the 0.0.0.0 address as the broadcast address and if you want to switch the broadcast address globally you can modify the config-register and reload the router.\
However, if you want to specify a different broadcast address to be used only on some interfaces, you can use the command:

```
R(config-if)# ip broadcast-address BROADCAST-ADDR
```
