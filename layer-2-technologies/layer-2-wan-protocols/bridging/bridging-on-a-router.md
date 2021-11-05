# Bridging on a router

## Bridging

Transparent Bridging is the default operational mode of switches. They bridge between interfaces and switch between them without modifying any data in the frames.\
Routing is the default operation mode of routers. They route between interfaces, and when doing this they modify the packets (Source and Destination MAC, TTL, etc)\
To enable routing, you use:

```
Sw(config)# ip routing
```

This command is default on routers, but must be manually entered on switches.\
Now, to enable bridging on routers, you have a few options:

### Transparent Bridging

By default, a router can only route a protocol. It can also bridge it, but it cannot do both at the same time. In order to enable transparent bridging, you will have to disable routing:

```
R(config)# no ip routing
```

By disabling routing, we will not be able to route between interfaces, but the router can still act as an IP host. Besides bridging interfaces, you can also set an IP address on the physical interfaces to make the router accessible in that subnet. One workaround to having routing disabled is to set the same IP address on all interfaces that are part of the bridge, thus making the router accessible on all interfaces.

### CRB – Concurrent Bridging and Routing

CRB is a way of performing both bridging and routing at the same time on a router. In this way, some interfaces can be used for routing, while others will be bridged, but they cannot do both at the same time.\
To enable CRB, leave ip routing on and set:

```
R(config)# bridge crb
```

Next you will have to define [how interfaces are bridged](broken-reference).

### IRB – Integrated Bridging and Routing

IRB is an upgrade from CRB, where you can do both routing and bridging at the same time. When using IRB, you can define a Bridged Virtual Interface (BVI) that will be able to route the traffic on the bridged interfaces, just like an SVI on a switch.\
To enable irb, leave ip routing on and set:

```
R(config)# bridge irb
```

You will still have to define [how interfaces are bridged](broken-reference).\
Then, in order to set up the BVI interface, first define what protocols can be routed:

```
R(config)# bridge BRIDGE route {ip|clns}
```

Then, you can use the BVI interface, just like a normal routed interface:

```
R(config)# interface bvi BRIDGE
R(config-if)# ip address IP-ADDR MASK
```

## Bridging interfaces

After defining what type of bridging to use, you have to define how the interfaces are bridged. Follow these steps:

### Define the Spanning Tree Protocol

First we need to define what type of Spanning Tree will run on this bridge:

```
R(config)# bridge BRIDGE protocol {ieee|dec|ibm|vlan-bridge}
! ieee = 802.1D
```

### Assign interfaces to the bridge

```
R(config-if)# bridge-group BRIDGE
```

At this point, you should check that the interfaces in this group act as a Layer 2 Switch:

```
R5#sh bridge group
Bridge Group 1 is running the IEEE compatible Spanning Tree protocol
   Port 4 (FastEthernet0/0) of bridge group 1 is forwarding
   Port 5 (FastEthernet0/1) of bridge group 1 is forwarding
```

To see a list of MAC addresses learned on each bridge, use:

```
R# show bridge [BRIDGE]
```

and to see the status of the spanning tree, use:

```
R# show spanning-tree [brief]
```

## Bridging with Frame Relay interfaces

### Different Encapsulations

When connecting an Ethernet interface (R1-Fa0/1) to another Ethernet interface(R2-Fa0/1) that is part of a bridge(R2-BVI1), you can ping from one side (R1-Fa0/1) to the other (R2-BVI1). When connecting a Frame Relay interface(R1-S1/0) to another Frame Relay interface(R2-S1/0) that is part of a bridge(R2-BVI1), you will encounter a problem with encapsulation. On one side (R1-S1/0) sends packets with Frame Relay encapsulation, and on the other side (R2-BVI1), packets with an Ethernet-ARPA encapsulation are expected. The only solution here is to create one bridged interface on each side (R1-BVI1, R2-BVI1).

### Mapping

Another problem arises when using multipoint frame-relay interfaces. On a point-to-point interface that is part of the bridge, the router will use the DLCI assigned to the interface to send data. But on multipoint interfaces, an explicit mapping is required for each DLCI:

```
R(config-if)# frame-relay map bridge DLCI [broadcast]
!broadcast needed to send BPDUs
```

### Switching between spokes

It won’t work. If the interface on the hub is a multipoint interface, then it will consider the spokes as two hosts connected on the same bridge port. In this case, it will drop the frame, considering that the frame sent by the source also reached the destination. There are 2 options here: either use a tunnel or force the traffic to go on another link by manipulating the spanning-tree (provided you have another link).
