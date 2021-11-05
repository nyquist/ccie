# Switchport Traffic Control

## Strom Control

The Storm Control feature, will disable the interface as soon as a specific threshold is passed. The threshold is measured every 1 second. The threshold can represent the amount of broadcast, multicast or unicast traffic and it can configured with:

```
! As a percentage of bandwidth:
Sw(config-if)# storm-control {broadcast|multicast|unicast} level LEVEL [LEVEL-LOW]
! As bandwidth in bps:
Sw(config-if)# storm-control {broadcast|multicast|unicast} bps BPS [BPS-LOW]
! As packets per seconds:
Sw(config-if)# storm-control {broadcast|multicast|unicast} pps PPS [PPS-LOW]
```

All traffic on the interface will be blocked when the rising threshold is passed. It will be resumed when traffic falls under the falling threshold.\
L2 multicast traffic used for control (BPDU, CDP) is not affected, but L3 multicast control traffic (routing protocols) is affected by this feature.\
The interface can be shutdown or it can generate a SNMP trap when the threshold is passed:

```
Sw(config-if)# storm-control action {shutdown|trap}
```

The suppression levels can be monitored with:

```
Sw# show storm-control [INTERFACE] [broadcast|multicast|unicasts]
```

## Small Frames

Frames smaller than 67 bytes are not counted by storm-control, but a similar mechanism can be enabeld:

```
! 1. Enable globally:
Sw(config)# errdisable detect cause small-frame
! 2. Enable per interface:
Sw(config-if)# small-violation-rate PPS
```

When the PPS threshold si passed, the port is errdisabled.

## Protected Ports

When 2 ports are defined as protected in a VLAN, they are completly isolated and cannot exchange traffic between them. They can exchange frames with other non-protected ports. It is similar to a private VLAN implementation, but only hardware switched packets are affected. Process switched packets are not affected by this feature.\
To define a proteced port, use:

```
Sw(config-if)# switchport protected
```

## Port Blocking

Port Blocking can be used to disable flooding of multicast, broadcast or unknown unicast from one port to others.

```
Sw(config-if)# switchport block [multicast|unicast]
```
