# IPv6 Multicast

## IPv6 Multicast Addressing FF00::/8

Multicast addresses represent a group of interfaces, just like in IPv4 multicast. The format of a multicast address is:

```
| 8 bits | 4b | 4b |     112 bits   |
+--------+----+----+----------------+
|11111111|0RPT|SCOP|    Group ID    |
+--------+----+----+----------------+
```

Flags:\
The R Flag is used for addresses with an embedded RP.\
The P Flag indicates a multicast address that is assigned based on the network prefix.\
The T Flag is set to 0 if the address is a well-known multicast (assigned by IANA) or 1 if the address is an administratively assigned address

The SCOP field (4 bits, 16 values) contains the scope of the multicast group. Available values are:

* **1**: Interface Local – spans only on a single interface and is used for loopback multicast
* **2**: Link Local – spans on a single link (point-to-point or multi-access)
* **4**: Admin Local – administratively defined
* **5**: Site Local – spans a single site
* **8**: Organizational Local – multiple sites belonging to a single organization
* **E**: Global

IANA keeps the [list of permanently assigned multicast addresses](https://www.iana.org/assignments/ipv6-multicast-addresses/ipv6-multicast-addresses.xml). The most used are:

| Address  | Description       |
| -------- | ----------------- |
| FF02::1  | All Nodes         |
| FF02::2  | All Routers       |
| FF02::5  | All OSPF Routers  |
| FF02::6  | All OSPF DRs      |
| FF02::9  | All RIP Routers   |
| FF02::A  | All EIGRP Routers |
| FF02::D  | All PIM Routers   |
| FF02::12 | All VRRP Routers  |
| FF02::16 | All MLD Routers   |

Other Addresses, like NTP are assigned by IANA but have a variable scope. The address assigned is FF0X::101, where X can be any of the scopes seen earlier.

## IPv6 Layer 2 Addresses

### Ethernet

The frames that carry multicast traffic have the MAC address of the sender as the source and a special address derived from the Layer 3 destination IPv6 as the destination.\
L2 Multicast addresses start with **33:33** followed by the last 32 bits in the IPv6 address.

## MLD

MLD (Multicast Listener Discovery Protocol) replaces IGMP for IPv6 multicast routing. MLDv1 is similar to IGMPv2, while MLDv2 is similar to IGMPv3 (supports Source Specific Multicast).\
To see the interfaces that run MLD, use:

```
R# show ipv6 mld interface
```

## PIM

The default version of PIM used on Cisco routers, PIMv2 also supports IPv6. To enable IPv6 multicast routing, use:

```
R(config)# ipv6 multicast-routing
```

This command also enables PIM on all IPv6 enabled interfaces.\
Only PIM-SM is supported for IPv6 and it works similar to IPv4 PIM.

### PIM Tunnels

Every router creates a tunnel to the RP that it uses to unicast PIM Register messages. These tunnels are automatically created. You can see them using:

```
R# show ipv6 pim tunnels
```

### Rendezvous Point

RP can be configured statically or using BSR. Auto-RP is not supported for IPv6.\
To see a list of configured RPs, use:

```
R# show ipv6 pim range-list
```

#### **Static RP**

To configure a static RP, use:

```
R(config)# ipv6 pim rp-address IPV6-ADDR [bidir]
```

#### **BSR**

To configure a BSR candidate, use:

```
R(config)# ipv6 pim bsr candidate bsr IPV6-ADDR [HASH] [priority PRI]
```

To configure a RP candidate, use:

```
R(config)# ipv6 pim bsr candidate rp IPV6-ADDR [group-list ACL] [priority PRI] [bidir]
```

#### **Embedded RP**

A new way to specify the RP is to embedded it into the multicast group address. The format of the group address is:

```
| 32 bits |      64 bits      | 32 bits  |
+---------+-------------------+----------+
|FF7S:0iLL|     RP Prefix     | Group ID |
S  = SCOPE
i  = Interface ID (4 bits)
LL = prefix Length. Ex: /64 => 0x40
RP Address: RP_Prefix::i/LL
```

E.g. If you want to use a RP with the address 2002:CC1E::9, you should assign the following address to the group: FF7x:0940:2002:CC1E::1. Disecting this address, we get:

* FF7 – marks the embedded RP type
* x – set it to the scope. E.g: for global scope: FF7E:940:2002:CC1E::1. For site local scope: FF75:940:2002:CC1E::1
* 9 – interface ID
* 40 – Hex version of the prefix lenght 0x40 = 64
* 2002:CC1E – prefix of the RP address
* ::1 – ID of the multicast group

Since the RP can be determined on each packet, there is no need for a RP protocol. Still, the RP needs to be configured with its own IP Address as the RP in order to accept registrations:

```
R(config)# ipv6 pim rp-address IPV6-ADDR
```

## Static IPv6 mroutes

IPv6 static multicast routes are added just like normal unicast routes, but with the **multicast** keyword at the end:

```
R(config)# ipv6 route IPV6-SOURCE {IPV6-PREVIOUS-HOP| INCOMING-INTERFACE multicast}
```
