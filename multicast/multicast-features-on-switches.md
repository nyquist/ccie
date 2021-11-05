# Multicast features on switches

## IGMP Snooping

### Enabling IGMP Snooping

Normally, a switch should forward multicast frames out all ports. It will forward the traffic to all receivers, but it will also forward it to many hosts that don’t need it. To optimize this behavior, Cisco routers implement IGMP Snooping. With IGMP Snooping, a switch listens to IGMP JOINs and LEAVEs to know which ports should receive traffic for a certain multicast destination.\
IGMP Snooping is on by default for all VLANs, If disabled, you can re-enable it with:

```
Sw(config)# ip igmp snooping [vlan VLAN-ID]
```

Snooping works not only with IGMP, but also with CGMP, PIM or DVMRP. If you want to limit learning only for one of these protocols, use:

```
Sw(config)# ip igmp snooping [vlan VLAN-ID] mrouter learn {cgmp | pim-dvmrp}
```

For IGMPv2 you can enable immediate leave support. This is useful when only one host is connected to a port. Therefor there is no need to leave the port in forwarding state if the only host doesn’t need the multicast traffic anymore:

```
R(config)# ip igmp snooping vlan VLAN-ID immediate-leave
```

For IGMPv1 and v2 you can enable Report Suppression. That is when an IGMP Query is sent, only the first Report is allowed, if other reports are being sent by other hosts they will be dropped, because the router will not use them. This feature is on by default, but if disabled you can enable it wit:

```
R(config)# ip igmp snooping report-suppression
```

### Configuring host ports

Usually a switch will detect a port where a PIM router is connected. However, you can configure it statically as a Multicast Router port with:

```
Sw(config)# ip igmp snooping vlan VLAN-ID mrouter interface INTERFACE
```

IGMP hosts are learned by snooping on IGMP messages they send, but you can also add static members to the list of forwarding ports:

```
Sw(config)# ip igmp snooping vlan VLAN-ID static GROUP-ADDR interface INTERFACE
```

To monitor, use:

```
Sw# show ip igmp snooping
```

### IGMP Querier

If there is no router on the VLAN that can send the queries you can enable the IGMP Querier function on the Switch. To do this, you should have a SVI with an IP Address running in that VLAN. If there’s no address on the SVI then a Global IGMP Querier address can be used. To enable IGMP Querier function, use:

```
R(config)# ip igmp snooping querier
R(config)# ip igmp snooping querier address IP-ADDR
! Define a global address to be used by the Querier, if the SVI doesn't have one
```

## Filtering IGMP

### IGMP Profiles

To filter IGMP messages for some multicast groups, first define an IGMP Profile:

```
Sw(config)# ip igmp profile PROFILE-NUMBER
Sw(config-igmp-profile)# permit|deny
!default: deny
Sw(config-igmp-profile)# range GROUP-ADDRESS
```

Then, apply the profile on a L2 interface:

```
Sw(config-if)# ip igmp filter PROFILE-NUMBER
```

### Max Groups

You can also set a maximum number of groups an interface can JOIN, using:

```
Sw(config-if)# ip igmp max-groups MAX
```

When the maximum number of groups is reached, a new IGMP REPORT is dropped by default, but it can also replace an existing GROUP if configured:

```
Sw(config-if)# ip igmp max-groups action {deny | replace}
```

## CGMP – Cisco Group Management Protocol

CGMP is a used between Cisco routers and switches only. With CGMP, a router is able to send information gathered through IGMP to the switch in order to optimize the CAM table for multicast traffic.\
To enable CGMP on a router, use:

```
R(config-if)# ip cgmp
! If used on a L3 switch, apply the config on a L3 interface
```

CGMP messages are sent only by the routers to a well known MAC address (0100.0CDD.DDDD) that the switches listen to and forward out all ports (in order to reach other switches). Through CGMP, a router informs the switches that a specific host (identified by its MAC Address) wants to receive multicast traffic for a specific group (identified by its L2 address 0100.5Exx.xxxx). The switches will then add entries into their CAM tables so that when they receive the multicast traffic (destined for MAC address 0100.5Exx.xxxx) it will only be forwarded out ports where the hosts that joined that group are located (by searching the host MAC address into existing CAM table). This way, there’s no unnecessary traffic sent. A similar mechanism exists to remove entries.

## MVR – Multicast VLAN Registration

MVR feature is somewhat similar to the voice VLAN feature. All multicast traffic for a Group Address is bound to a single VLAN, and receivers send IGMP JOINs to ask for traffic from the multicast VLAN.\
First, we must configure the switch to support MVR:

```
Sw(config)# mvr
! Enables MVR
Sw(config)# mvr group GROUP-ADDR [COUNT]
! the switch will put traffic for this address in the MVR VLAN
! The COUNT will allow MVR to work for several contiguous Group Addresses
Sw(config)# mvr vlan VLAN-ID
! sets the VLAN used for MVR traffic
Sw(config)# mvr mode {dynamic|compatible}
! Dynamic - forwards IGMP Joins to the router
! Compatible - doesn't forward IGMP Joins. The router must be configured to force multicast traffic downstream (default)
```

To configure the source ports:

```
Sw(config-if)# mvr type source
! Source ports must be part of the Multicast VLAN
```

To configure receivers:

```
Sw(config-if)# mvr type receiver
Sw(config-if)# mvr vlan VLAN-ID group [GROUP-ID]
! Statically adds ports to the MVR, without IGMP JOINs
Sw(config-if)# mvr immediate
! Enables Immediate Leave for MVR
```

To monitor, use:

```
Sw# show mvr [members|interface] ...
```
