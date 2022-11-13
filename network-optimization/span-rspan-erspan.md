# SPAN, RSPAN, ERSPAN

SPAN allows the traffic on one swithcport to be copied on another switchport. A SPAN session needs to be created where you define source ports and destination ports. Alternatively you can configure a source VLAN where all ports in the VLAN become the source ports. You can't mix between source ports and source VLANs

## Local SPAN

Local SPAN (Switched Port ANalyzer) is an association of source ports or source VLANs with one or more destination ports on the same switch.

To configure a local SPAN session use these commands:

```
Sw(config)# monitor session SESSION-ID source {interface SRC-INTF-ID | vlan SRC-VLAN-ID} [rx|tx|both]
! rx - filter only incoming packets
! tx - filter only outgoing packets
! both - don't filter incoming or outgoing packets (default)
Sw(config)# monitor session SESSION-ID destination interface DST-INTF-ID
```

The number of destination ports is platform dependent.&#x20;

To monitor the current SPAN sessions, use

```
Sw# show monitor
```

## RSPAN

RSPAN (Remote SPAN) allows the source and the destination to be on different switches. The way this is achived is by using a RSPAN VLAN to carry SPAN traffic between switches.

To configure RSPAN, use these commands on switches that contain source ports:

```
Sw1(config)# vlan VLAN-ID
Sw1(config-vlan)# remote-span
Sw1(config-vlan)# exit
Sw1(config)# monitor session SESSION-ID source interface SRC-INTF-ID [rx|tx|both]
Sw1(config)# monitor session SESSION-ID destination remote vlan VLAN-ID

```

and these commands on switches that contain destination ports

```
Sw2(config)# vlan VLAN-ID
Sw2(config-vlan)# remote-span
Sw2(config-vlan)# exit
Sw2(config)# monitor session SESSION-ID source remote vlan VLAN-ID
Sw2(config)# monitor session SESSION-ID destination interface DST-INTF-ID
```

Obviously, the RSPAN vlan needs to be configured on all switches. To monitor the RSPAN sessions use these commands:

```
Sw# show monitor
Sw# show vlan remote-span
```

## ERSPAN

ERSPAN (Encapsulated Remote SPAN) takes the concept further and encapsulates the source traffic in GRE, allowing it to be routed over a Layer3 network to the destination ports.

To configure ERSPAN source devices, use:

```
Sw1(config)# monitor session SESSION-ID type erspan-source
Sw1(config-mon-erspan-src)# source interface SRC-INTF-ID
Sw1(config-mon-erspan-src)# destination
Sw1(config-mon-erspan-src-dst)# erspan-id ERSPAN-ID
Sw1(config-mon-erspan-src-dst)# ip address DST-IP
Sw1(config-mon-erspan-src-dst)# origin ip address SRC-IP
```

To configure ERSPAN destination device, use:

```
Sw1(config)# monitor session SESSION-ID type erspan-destination
Sw1(config-mon-erspan-dst)# destinaation interface SRC-INTF-ID
Sw1(config-mon-erspan-dst)# source
Sw1(config-mon-erspan-dst-src)# erspan-id ERSPAN-ID
Sw1(config-mon-erspan-dst-src)# ip address DST-IP
```
