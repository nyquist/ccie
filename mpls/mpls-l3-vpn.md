# MPLS L3 VPN

This article assumes the “provider” network already has an IGP in place and that the LDP is configured to advertise label bindings between LSRs. Check [MPLS 101](mpls-101.md) on how to do that.

## Verify LDP is working within provider network

One common mistake when configuring L3 MPLS VPN appears when using OSPF as the IGP. L3 MPLS VPN uses 2 stacked labels when sending customer traffic from one PE to another. The outer most label identifies traffic for the PE and should be popped before reaching the PE, through PHP. When using OSPF, loopback addresses are advertised by default as /32 prefixes, regardless of the actual subnet. However, in MPLS they will be advertised with their subnet. If the loopback address is different then /32 the PE router will advertise via LDP a network with, let’s say, /24 (because the Connected route will be in it’s routing table), while it’s LDP neighbor will advertise a /32 route (known via OSPF). They will not agree upon a label, and the penultimate router will consider that it should route the packets as IP, not as MPLS. You can spot such mistakes when looking in the MPLS forwarding table with:

```
R# show mpls forwarding-table
```

Prefixes that point to the PE’s loopback should have the operation POP. If the operation is UNTAGGED, then there is a mismatch between the advertisement of the PE and the penultimate router.\
Usually, in a working MPLS network, you should only see UNTAGGED on egress routers to destinations that do not run MPLS.

## Create MPLS Tunnels between PE routers

Before MPLS, a provider would run eBGP connections with its customers, and iBGP connections inside the provider network.\
Using iBGP requires an iBGP full mesh (administrative nightmare) or the use of route-reflectors or confederations (it may still be hard to configure in large networks and requires all routers to speak BGP)\
The more elegant solution is to use MPLS between the PE routers and to make iBGP connections only between the PE routers.

If we do not use MPLS between the PE routers, all traffic will be black-holed when it reaches the first non-BGP speaker, which will not know how to route the packet. If we use MPLS, the PE routers will encapsulate the traffic that must pass through the network and out to another AS in MPLS packets that point toward the egress PE router. This router will know how to forward the traffic using destination IP. There are some rules though:

1.  iBGP connections between routers running MPLS must be brought up using the loopback addresses

    ```
    PE-RTR(config-router)# neighbor PE-ROUTER update-source LOOPBACK-ADDRESS
    ```

    If we use the interface IP to establish the neighbor relationship, PHP will kick in and it will POP the label before it reaches the last PE router, and the provider router in the MPLS cloud will not know about the BGP destination. You can also use explicit NULL advertisements to avoid this.
2.  set the next-hop self for iBGP connections

    ```
    PE-RTR(config-router)# neighbor PE-ROUTER next-hop-self
    ```

    Normally, eBGP learned routes are advertised with an unchanged NEXT\_HOP value so we must first change this. Also, NEXT\_HOP value must match the egress PE router’s loopback address because this is used in LDP. Both requirements are solved using the above command.

You can also use route-maps to change the next-hop to match the LDP Router ID.

## Create the Layer 3 MPLS VPNs

### Define VRF for each customer

Using VRFs, you can have separate routing tables for each customer. Interfaces are assigned to a VRF, but by default they all belong to the global routing table.

1.  Define VRF:

    ```
    PE-RTR(config)# ip vrf VRF-NAME
    ```
2.  Assign interfaces to the VRF:

    ```
    R(config-if)# ip vrf forwarding VRF-NAME
    ! must re-set the IP Address on the interface
    ```
3.  Verify:

    ```
    PE-RTR# show ip vrf [detail]
    PE-RTR# show ip route vrf {VRF-NAME|*}
    PE-RTR# {ping|telnet|traceroute} vrf VRF-NAME IP-ADDR
    PE-RTR# show mpls ip bindings
    ```

When you use VRFs without any MPLS or MP-BGP involved, it is called a VRF Lite. It’s simply the possibility to use separated/virtualized routing tables in a single router.

### Create VPNv4 routes and advertise them in MP-BGP

#### **RD – Route Distinguisher**

The PE routers will learn the customer routes via static routes, an IGP routing protocol or eBGP. These routes are then redistributed into MP-BGP as VPNv4 routes. A VPNv4 is created by adding a prefix to the customer route in order to make it unique in the MP-BGP table. The prefix is called RD(Route Distinguisher) and is usually in the format AS:NN or IP-ADDR:NN. The full VPNv4 prefix is RD:CUSTOMER-PREFIX/MASK-LEN. (Or AS:NN:CUSTOMER-PREFIX/MASK-LEN)\
To configure a VRF and set it up with an RD, use the following commands:

```
PE-RTR(config)# ip vrf VRF-NAME
PE-RTR(config-vrf)# rd RD
```

You will most likely need to add interfaces to the VRF:

```
PE-RTR(config-if)# ip vrf forwarding VRF-NAME
```

Assigning an interface to a VRF will remove its configured IP Address!

#### **RT – Route Target**

When routes are redistributed from the VRF into MP-BGP, they also get an extended Community attached, the Route Target Export. Similarly, on the other PE routers, routes are redistributed from MP-BGP into the customer VRF only if they have the same Route Target value as the VRF’s Route Target Import.\
A VPNv4 route can have more Route Targets assigned to it. As long as one of them matches one of the Route Target Import values on the other PE’s VRF, it will be redistributed into the customer VRF. These routes are only known by the PE routers. Inside the MPLS cloud the switching is done based on MPLS labels only

To setup the Route Targets specific for a VRF, use:

```
PE-RTR(config)# ip vrf VRF-NAME
PE-RTR(config-vrf)# route-target export EXPORT-RT
PE-RTR(config-vrf)# route-target import IMPORT-RT
! or in one single command if you use the same value:
PE-RTR(config-vrf)# route-target both EXPORT-IMPORT-RT
```

For intranet VPNs, The export RT should match the import RT on the other PEs and vice-versa. For more complex scenarios (extranet VPNs) you can use any combination of Route Targets.

#### **Connect PE routers via MP-BGP and enable VPNv4 address family**

To connect 2 or more customer sites, you need to create iBGP sessions between their respective PE routers. Otherwise, they won’t be able to exchange VPNv4 routes. To do this, use the following steps:

```
R(config)# router bgp AS-NUMBER
PE-RTR(config-router)# neighbor PE-LOOPBACK-ADDR remote-as AS-NUMBER
PE-RTR(config-router)# neighbor PE-LOOPBACK-ADDR update-source LOOPBACK-INTERFACE
```

The PE routers can connect via BGP in a Full Mesh, Hub and Spoke or any other architecture that is needed

To exchange VPNv4 routes in MP-BGP, you need to enable it in the BGP process:

```
PE-RTR(config-router)# address-family vpnv4
PE-RTR(config-router-af)# neighbor PE-LOOPBACK-ADDR activate
! activates the vpnv4 address-family for this neighbor
PE-RTR(config-router-af)# neighbor PE-LOOPBACK-ADDR send-community extended
! allows sending RT as an extended community
```

During the VPNv4 exchange, PE routers also agree on a “VPN label”. This label will be

#### **Redistribute customer routes into MP-BGP**

For VPNv4 routes to be created, you need to have IPv4 routes in the BGP process specific to the customer VRF. To do this, you must enable the address family for that specific VRF and redistribute the VRF routes into MP-BGP.

```
! On the PE router:
PE-RTR(config)# router bgp AS-NUMBER
PE-RTR(config-router)# address-family ipv4 vrf VRF-NAME
PE-RTR(config-router-af)# redistribute {static | rip | ospf PROC | eigrp AS | bgp} ... 
```

### Advertise customer routes

Since MP-BGP will advertise the customer routes from PE to PE, we need a way to advertise customer routes from CE to PE and redistribute them in MP-BGP, as well as a way to get the routes from MP-BGP and advertise them from PE to CE. You can use static routes or a dynamic IGP to achieve this.

{% tabs %}
{% tab title="Static Routes" %}
On the PE router:

```
! Customer Destination pointing to the CE Router:
PE-RTR(config)# ip route vrf VRF-NAME DESTINATION MASK NEXT-HOP-ADDR
```

Then redistribute the static or connected routes into MP-BGP:

```
PE-RTR(config)# router bgp AS-NUMBER
PE-RTR(config-router)# address-family ipv4 vrf VRF-NAME
PE-RTR(config-router-af)# redistribute static ...
PE-RTR(config-router-af)# redistribute connected ...
```

On the CE router, you can set a default gateway or more specific routes via the PE router.
{% endtab %}

{% tab title="RIP" %}
VRF aware RIP is configured inside the global RIP process, and each command can be configured for each VRF (network, version, auto-summary, etc):

```
PE-RTR(config)# router rip
PE-RTR(config-router)# address-family ipv4 vrf VRF-NAME
PE-RTR(config-router-af)# network NETWORK-ADDR
PE-RTR(config-router-af)# ...
! Redistribute routes from BGP into VRF RIP:
PE-RTR(config-router-af)# redistribute bgp...
```

Redistribute each VRF RIP into MP-BGP:

```
R(config)# router bgp AS-NUMBER
R(config-router)# address-family ipv4 vrf VRF-NAME
R(config-router-af)# redistribute rip ...
```

On the CE routers, RIP is configured normally.
{% endtab %}

{% tab title="EIGRP" %}
VRF aware EIGRP is configured inside one global EIGRP process, but each command can be configured for each VRF (network, auto-summary, etc) and the EIGRP AS-NUMBER is not inherited for each VRF, instead it must be manually configured:

```
R(config)# router eigrp GLOBAL-AS
R(config-router)# address-family ipv4 vrf VRF-NAME
R(config-router-af)# network NETWORK-ADDR WILDCARD
R(config-rotuer-af)# autonomous-system VRF-AS
R(config-router-af)# ...
! Redistribute routes from BGP into VRF EIGRP:
R(config-router-af)# redistribute bgp ... 
```

Redistribute each EIGRP VRF into MP-BGP:

```
R(config)# router bgp AS-NUMBER
R(config-router)# address-family ipv4 vrf VRF-NAME
R(config-router-af)# redistribute eigrp VRF-AS...
```

On the CE routers, EIGRP is configured normally.
{% endtab %}

{% tab title="OSPF" %}


When using OSPF, between the PE and the CE, you must use different OSPF processes for each VRF, in addition to the OSPF process used in the MPLS cloud. Also, each OSPF process must have a different Router ID.

```
R(config)# router ospf PROC vrf VRF-NAME
R(config-router)# router-id ROUTER-ID
R(config-router)# network NETWORK-ADDR WILDCARD area AREA
R(config-router)# ...
! Redistribute routes from BGP into VRF EIGRP:
R(config-router)# redistribute bgp AS-NUMBER ... 
```

You will have to redistribute each OSPF VRF into MP-BGP:

```
R(config)# router bgp AS-NUMBER
R(config-router)# address-family ipv4 vrf VRF-NAME
R(config-router-af)# redistribute ospf PROC vrf VRF-AS
```

On the CE routers, OSPF is configured normally.

**Domain ID**

The OSPF Domain ID is usually the same as the process ID, written in IP-ADDRESS format (Ex: 0.0.0.1).If the Domain ID differs between PE customer VRFs, the routes are redistributed as external, but if the domain ID is the same, OSPF internal routes from other sites are redistributed as InterArea (O IA) routes.\
You can verify the Domain ID with

```
R# show ip ospf [PROC-ID] | i Domain
   Domain ID type 0x0005, value 0.0.0.1
```

To change it, use:

```
R(config)# router ospf PROC-ID vrf VRF-NAME
R(config-router)# domain-id DOMAIN-ID
```

**Sham Links**

When running OSPF as the customer protocol, the following scenario may have unwanted results. Since the customer routes redistributed in OSPF from BGP appear as external or InterArea routes (as seen before), if the 2 sites have a backdoor connection that runs OSPF, then it is highly probable that we will learn the same routes over this backdoor connection as IntraArea routes (O). As you know, OSPF will always prefer IntraArea routes to InterArea and External Routes, so there is no way you can force the customer traffic to use the VPN rather than the backdoor (usually used for backup) route. Unless you use sham-links! Sham links are virtual links that are used to connect 2 customer PEs. By learning the routes over the sham link, they will be considered IntraArea (O) and may be preferred based on the cost to the routes learned via the backdoor links.\
To configure a sham link, we will need one additional /32 loopback on each PE. These loopbacks will not be advertised using OSPF into the customer VRF, instead they will be advertised into BGP. On the PEs we will configure:

```
PE1(config)# interface LO1
PE1(config-if)# ip address IP-ADDR1 255.255.255.25
PE1(config)# router bgp AS-NUMBER
PE1(config-router)# network NETWORK1 mask 255.255.255.255
PE1(config)# router ospf PROC-ID1 vrf VRF-NAME1
PE1(config-router)# area AREA-ID1 sham-link IP-ADDR1 IP-ADDR2 [cost COST]
! IP-ADDR1 is configured on PE1, and IP-ADDR2 si configured on PE2
```

and

```
PE2(config)# interface LO1
PE2(config-if)# ip address IP-ADDR2 255.255.255.25
PE2(config)# router bgp AS-NUMBER
PE2(config-router)# network NETWORK2 mask 255.255.255.255
PE2(config)# router ospf PROC-ID2 vrf VRF-NAME2
PE2(config-router)# area AREA-ID1 sham-link IP-ADDR2 IP-ADDR1 [cost COST]
```

You can verify the sham-link status using:

```
R#show ip ospf sham-links
```

Now if you look at the OSPF routes on the CE the InterArea (O IA) routes should be replaced by the IntraArea (O) routes. Of course, the metric of the routes received via the sham links should be better than via the backdoor link.


{% endtab %}
{% endtabs %}

### **Verify**

```
R# show ip bgp vpnv4 [VRF-NAME | all]
R# show ip bgp vpnv4 rd RD labels
R# show mpls ip binding
```

### How a packet moves in an MPLS VPN network

When the PE routers become peers, and exchange VPNv4 rotues, they also agree on a “VPN label” (or “BGP Label”)to be used for each prefix. This label will be at the bottom of the stack (BoS:=1) and will be attached to all packets that are intended for a specific prefix inside the customer network.\
At the same time, since the PE routers had their “next-hop self” configured, the VPNv4 routes received by each PE have their next hop set to the other PE’s IP address. This address is advertised inside the provider cloud and via LDP, son when adding the MPLS header to a packet coming/going to the customer network, the router must add 2 labels:

* Transport/IGP Label – as the label used to forward traffic towards the destination PE router
* VPN/BGP Label (BoS=1) – identifies the customer VPN when reaching the PE router

The Transport label is swaped as the packet goes through each LSR in the LSP. Usually, just before reaching the PE router, the Transport label is popped (due to PHP). Now, if a PE router has multiple VRF customers, how would it know where to forward the packet? IP information is not good enough, because address spaces can overlap. That’s why we used VPNv4. This is where the VPN label comes into play. It will help the router understand what VRF it is destined for. Since the label is agreed during VPNv4 route exchange, the label is bind to a specific VRF.\
Once this information is clear, the router can strip the MPLS labels and forward the packet based on its IP address.
