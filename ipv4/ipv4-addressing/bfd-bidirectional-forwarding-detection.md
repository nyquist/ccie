# BFD – Bidirectional Forwarding Detection

## What is BFD?

BFD stands for Bidirectional Forwarding Detection and it’s a protocol that is used for rapid detection of link failures when the line-protocol is still “up”. BFD is enabled on interface and creates a BFD session with the neighboring router (BFD Peer). Routing protocols such as EIGRP, OSPF and BGP support BFD and can rapidly detect a link failure without waiting for hold-timers to expire.

## Enabling BFD

To enable BFD per interface, use:

```
R(config)# interface INTERFACE
R(config-if)# bfd interval TX-MSEC min_rx RX-MSEC multiplier COUNT
! TX-MSEC = rate at which BFD packets are sent
! RX-MSEC = Min rate at which BFD packets are expected
! COUNT = how many packets can be missed before considering the link down
```

To verify BFD, use:

```
R# show bfd neighbors [detail]
```

## BFD Support for routing protocols

### EIGRP

To enable BFD support for EIGRP, use:

```
R(config)# router eigrp AS-NUMBER
R(config-router)# bfd {interface INTERFACE|all-interfaces}
```

To verify, use:

```
R# show ip eigrp interfaces ... [details] 
```

### OSPF

To enable BFD support for OSPF, there are 2 options:\
Option 1 is to enable OSPF support for BFD, one-by-one on each interface:

```
R(config)# interface INTERFACE
R(config-if)# ip ospf bfd
```

Option 2 is to enable OSPF support for all BFD interfaces and the disable it on unneeded interfaces:

```
R(config)# router ospf PROC-ID
R(config-router)# bfd all-interfaces
! Will enable BFD support for all interfaces
```

You can disable then OSPF support for BFD, per-interface, with:

```
R(config-if)# ip ospf bfd disable
```

To verify, use:

```
R# show ip ospf [details]
```

#### 2.3BGP

To enable BFD support for BGP, use:

```
R(config)# router bgp AS-NUMBER
R(config-router)# neighbor NEIGH-IP fall-over bfd
```

To verify, use:

```
R# show ip bgp neighbors
```
