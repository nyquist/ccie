# DMVPN 101

## DMVPN

DMVPN – Dynamic Multipoint VPN is a technology that uses IPSec, mGRE and NHRP to provide a dynamic VPN infrastructure.

DMVPN evolved in several phases as follows:

* **DMVPN phase 1**: Hub and Spoke – spokes only communicate via Hub
* **DMVPN phase 2**: Hub and spoke with spoke to spoke tunnels – spokes can create tunnels between themselves, but the hub is used to provide information on how to reach the spokes
* **DMVPN phase 3**: Hub and spoke with spoke to spoke tunnels – spokes can also provide reachability information so the role of the hub is reduced.

## GRE and NHRP

mGRE (multipoint GRE) generalizes the GRE concept by allowing multiple, dynamic GRE tunnels initiated from a single device.\
NHRP (Next Hop Resolution Protocol) is used to resolvee L3 to L2 address mapping in NBMA environments (like DMVPN or Frame Relay)\
NHRP operates by using servers (NHS) and clients (NHC). A client needs to register with at least one server, and the servers collect client addressing information in order to provide it to other clients when requested. A server can query another server if it’s missing information requested by clients.

### Enable mGRE

```
R(config)#interface TUNNEL-INT
R(config-if)#tunnel mode gre multipoint
R(config-if)#tunnel source INTERFACE-NAME
! You don't specify a tunnel destination with mGRE
R(config-if)#tunnel key TUNNEL-KEY
! all nodes must share the same TUNNEL-KEY
```

The NBMA address of the node will be the IP address of the tunnel source.\
When using EIGRP or RIP, split horizon should be disabled on the hub tunnel interface.

### Enable NHRP

```
R(config)#ip nhrp network-id NETWORK-ID
! All nodes must share the same NETWORK-ID
```

### Building the NHRP cache – mapping addresses

#### **Static mapping**

```
R(cisco)# ip nhrp map TUNNEL-DESTINATION-IP NODE-INTERFACE-IP
!TUNNEL-DESTINATION = ip address of the other node's tunnel interface
!NODE-INTERFACE-IP = ip address of the other node's physical interface (tunnel source)
```

Static mapping is required on the spokes to intiate the connection to the server.\
To map multicast traffic to a destination, use:

```
R(config)# ip nhrp multicast [NODE-INTERFACE-IP|dynamic]
! Usually, use NODE-INTERFACE-IP on the spokes
! Usually, use dynamic on the server
```

This is normally configured on the spokes, pointing to the hub and is required for some IGPs to work. (similar to Frame Relay’s broadcast mapping).

#### **Dynamic mapping**

An NHRP interface works both as server and client. A server can also have another server configured and it can forward requests to it’s known servers when it doesn’t have the complete mapping.\
The clients need to have the server destination configured:

```
R(config-if)# ip nhrp nhs [NHS-IP]
!the NHS-IP is the tunnel address of the server node
R(config-if)# ip nhrp holdtime [HOLDTIME]
```

Clients and servers can use an authentication key between them to prevent unwanted connections:

```
R(config-if)# ip nhrp authentication [AUTH-KEY]
```

A client sends a registration request to the server which includes it’s Tunnel interface address and it’s NBMA interface (physical) address.\
The server will create an entry in it’s NHRP cache and can use it to reply to other NHRP clients.\
To send multicast traffic to the spokes, the hub should also have this configured:

```
R(config)# ip nhrp map multicast dynamic
```

The hub can’t intitate connections to the spokes until they register, because it does’t know how to reach them.

#### **Check mapping**

To verify the NHRP cache, use:

```
R#show ip nhrp detail
10.0.123.2/32 via 10.0.123.2, Tunnel0 created 00:24:32, expire 00:00:30
Type: dynamic, Flags: authoritative unique registered used
NBMA address: 192.168.1.2
10.0.123.3/32 via 10.0.123.3, Tunnel0 created 00:26:43, expire 00:00:55
Type: dynamic, Flags: authoritative unique registered used
NBMA address: 192.168.1.3
```

The cache entries have some flags associated to them, which can be:

* **authoritative** = This entry was learned from the next hop server.
* **registered** = This entry was learned by the server from one of it’s clients.
* **implicit** = learned not from an NHRP request generated from the local router, but from an NHRP packet being forwarded or from an NHRP request being received by the local router
*   **unique** = If a client wants to register another entry which is already in the cache with this flag set, it will be rejected. Normally, all routers register their addresses with the “unique” flag. This can be disabled with:

    ```
    R(config-if)# ip nhrp registration no-unique
    ```
* **used** = This entry is used to switch traffic. See DMVPN Phase 2.

## IPSec protection

mGRE and NHRP can create a functional NBMA network, but the security is lacking, especially since this is used over a public/ISP interface.\
To easy deployment, the IPSec profile protection should be enabled. To do this, use:

```
R(config)# interface TUNNEL-INT
R(config-if)# tunnel protection ipsec profile IPSEC-PROFILE
```

The IPSEC-PROFILE is similar to an IPSEC-CRYPRO-MAP, but it lacks peer information and ACL to match interesting traffic. These will be derived from the mGRE configuration and the NHRP cache.\
See IPSEC article for details.

## DMVPN Phases

### Phase 1

Phase 1 uses static GRE tunnels on the spokes and a mGRE tunnel on the hub. It’s easy to configure but communication between spokes must go through the hub.\
RIP, EIGRP or OSPF can be used over DMVPN Phase 1.

### Phase 2

Phase 2 uses mGRE tunnels on spokes as well, allowing the creation of tunnels between spokes.\
EIGRP can be used over DMVPN phase 2, but the hub must not set itself as the next-hop, so the routers can intitiate the NHRP resolution and the creation of the mGRE tunnels.\
OSFP with broadcast network type can also be used.

```
R(config-if)# no ip next-hop self eigrp AS
```

To resolve the adjacency, CEF marks the tunnel destination as “glean” – meaning it will request NHRP data when traffic needs to be forwarded over that interface.\
When CEF is enabled, as long as traffic is sent to a node (which means a specific CEF entry is used), that entry doesn’t timeout. When the timer is less than 120 seconds, the CEF entry is marked as stale. If a new packet is switched over the stale entry, a new NHRP request is sent and the entry is refreshed. If no packet is switchd until the timer expires, the CEF entry times out becomes invalid.\
When CEF is not used, the entries expire based on the “used” flag associated with the map entries. When there’s no more traffic using that entry, it will timeout.

### Phase 3

The additions in phase 3 are that once the network is setup, CEF entries point to the hub address. When traffic reaches the hub, but is destined for another spoke, it sends a NHRP redirect which triggers a new NHRP request from the source spoke to the destination spoke. This request travels via the hub, but the reply is direct and foreces a rewrite of the CEF entry with the new spoke information. This operation is called a NHRP shortcut. To enable Phase 3, you should use the following commands:

```
R(config-if)# ip nhrp shortcut
! enables the NHRP shortcut - CEF rewrite
R(config-if)# ip nhrp redirect
! enables the NHRP redirect message
```
