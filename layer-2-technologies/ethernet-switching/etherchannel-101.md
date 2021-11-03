# EtherChannel 101

An etherchannel is a logical port that consists of multiple links bundled into a single logical link. To have a working etherchannel you must use static config or a negotiation protocol (LACP or PAgP). All ports in an EtherChannel must operate at the same speed and duplex. When an EtherChannel group is first created, it will follow the configuration of the first port that was added to the group. The following configurations must match between all ports in a group:

* Allowed VLAN List and Native VLAN
* Spanning-tree path cost and priority for each VLAN
* Spanning-tree PortFast setting

## Static Configuration

```
Sw(config)# channel-group PO-ID mode on
```

This configuration will work only if all ports in the channel-groups on both sides as configured like this. No negotiation takes place.

## PAgP

PAgP is a Cisco Proprietary protocol. Up to 8 ports of the same type can be configured for the same EtherChannel.

```
Sw(config)# channel-group PO-ID mode {auto|desirable} [non-silent]
```

* **auto** – enables PAgP passively – It will respond to PAgP packets, but will not start a negotiation.
* **desirable** – enables PAgP actively – it responds and starts a PAgP negotiation.
* _silent_ – by default, PAgP assumes the silent mode, when it considers that the other end doesn’t use PAgP so it will bring the portchannel Up, if the other end doesn’t respond to PAgP packets.
* **non-silent** – If set, then PAgP will wait for a response from the other before bringing up the PortChannel.

## LACP

LACP is a IEEE protocol. Up to 16 ports of the same type can be configured in an EtherChannel, but only 8 will be active, while the other 8 will be in standby. The software will determine which ports to use based on the lowest priority, which consists of:

1. LACP System Priority
2. System ID (Switch MAC)
3. LACP Port Priority
4. Port Number

The side with the lowest System Priority will decide what ports to use. To modify and verify the System Priority, use:

```
Sw(config)# lacp system-priority PRI
!Default: 32768
Sw# show lacp sys-id
```

To modify and veridy the LACP Port Priority, use:

```
Sw(config-if)# lacp port-priority PRI
!default: 32768
Sw# show lacp [PO#] internal
```

To set up an EtherChannel using LACP, use:

```
Sw(config)# channel-group PO-ID mode {active|passive}
```

* **passive** – enables LACP passively – It will respond to LACP packets, but will not start a negotiation.
* **active** – enables LACP actively – it responds and starts a LACP negotiation.

## Load Balancing

In order to decide on which link should traffic be forwarded out on a channel group, the switch uses a load balancing algorithm which takes the source or destination MAC or IP address of each packet and computes a hash value which is mapped to the port it should be forwarded on. As long as the input data is the same, the same port will be chosen for output.\
To set the load-balance method, use:

```
Sw(config)# port-channel load-balance {dst-ip|dst-mac|src-dst-ip|src-dst-mac|src-ip|src-mac}
!Default: src-mac
```

To verify, use:

```
Sw# show etherchannel load-balance
```

The hash is fixed length and most platforms use a 3 bit size. This creates some unbalance in utilization as the ports are mapped to the possible hash values:

| Ports in EtherChannel | Hash assignment        | Ratio           |
| --------------------- | ---------------------- | --------------- |
| 2                     | 1\|2\|1\|2\|1\|2\|1\|2 | 4:4             |
| 3                     | 1\|2\|3\|1\|2\|3\|1\|2 | 3:3:2           |
| 4                     | 1\|2\|3\|4\|1\|2\|3\|4 | 2:2:2:2         |
| 5                     | 1\|2\|3\|4\|5\|1\|2\|3 | 2:2:2:1:1       |
| 6                     | 1\|2\|3\|4\|5\|6\|1\|2 | 2:2:1:1:1:1     |
| 7                     | 1\|2\|3\|4\|5\|6\|7\|1 | 2:1:1:1:1:1:1   |
| 8                     | 1\|2\|3\|4\|5\|6\|7\|8 | 1:1:1:1:1:1:1:1 |

Other platforms use 8 bits for hashing which smooth out some of the unbalance

## Monitor

```
Sw# show ethernchannel [PO-ID] {summary|detail|...}
Sw# show pagp ...
Sw# show lacp ...
```
