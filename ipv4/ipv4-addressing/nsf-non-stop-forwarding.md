# NSF – Non Stop Forwarding

## What is NSF

NSF is a feature that allows routers to keep on forwarding traffic (non stop forwarding) even in the event of a restart.\
This is done by separating the control and the data plane, having one process involved in building the routing table and another process in forwarding the packets.\
This feature takes advantage of CEF which updates the line cards with the information from FIB.\
In order for NSF to work, routers must be NSF-capable or NSF-aware. A NSF-capable router is a router that can perform restarts without disrupting packet forwarding, while a NSF-aware router understands NSF-specific signaling from NSF-capable routers.\
NSF requires these additional features because, while performing restart, a router will not be able to send Hellos to its peers. This normally should result in the neighbor relationship being dropped, routes being lost, packets being dropped.\
Routers exchange NSF capabilities information with each other and when a NSF-capable router performs restart, its NSF-aware peers will change the default behavior in order to prevent breaking the neighbor relationship.\
NSF-aware routers are also called NSF-helpers because they will help a router performing a NSF restart to re-sync with the network as soon as possible.

## NSF Support

### EIGRP

To configure NSF supports for EIGRP, use:

```
R(config)# router eigrp AS-NUMBER
R(config-router)# address-family ipv4 unicast [vrf VRF] AS-NUMBER
R(config-router-af)# nsf
! Disabled by default
```

To modify the EIGRP NSF timers, use:

```
R(config-router-af)# timers graceful-restart purge-time SEC
! Default: 240 sec
R(config-router-af)# timers nsf converge SEC
! Default: 120 sec
R(config-router-af)# timers nsf route-hold SEC
! Default: 240 sec
R(config-router-af)# timers nsf signal SEC
! Defualt: 20 sec
```

### OSPF

Cisco routers support 2 NSF modes, a Cisco-proprietary and an IETF-open-standard version (RFC 3623).

**2.2.1 Cisco Mode**

To configure NSF in Cisco Mode:

```
R(config)# router ospf PROC-ID
R(config-router)# nsf cisco enforce global
! Enables a router to be NSF-capable. Disabled by default
R(config-router)# nsf helper [disable]
! Enables a router to be NSF-aware. Enabled by default
```

**2.2.2 IETF Mode**

To configure NSF for IETF Mode:

```
R(config)# router ospf PROC-ID
R(config-router)# nsf ietf [restart-interval SEC]
! default: 120 sec
R(config-router)# nsf ietf helper [disable]
!NSF-aware mode is enabled by default.
```

Strict LSA Checking is a feature of the IETF NSF mode for OSPF. When a router acts as a NSF-aware router (NSF-helper) it will stop the nsf-helper process if it detects a change of a LSA that would be forwarded to the restarting router. To enable this feature, use:

```
R(config-router)# nsf ietf helper strict-lsa-checking
! disabled by default
```

**2.2.3 Verify OSPF NSF**

To verify how OSPF works with NSF, you can use:

```
R# show ip ospf neighbors
R# show ip ospf nsf
R# debug ospf nsf [detail]
```

#### 2.3 BGP

To enable NSF for BGP, use:

```
R(config)# router bgp AS-NUMBER
R(config-router)# bgp graceful-restart [restart-time SEC | stalepath-time SEC] 
! Default restart-time : 120 sec
! Default stalepath-time: 360 sec
```

This command is enabled per address-family. You can enable it for all address families with:

```
R(config-router)# bgp graceful-restart all
```

To verify:

```
R# show ip bgp neighbors [NEIGH-IP]
```
