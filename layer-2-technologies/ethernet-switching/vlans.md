# VLANs

## Supported VLANs

Supported VLANs: 1-4094

```
Sw(config)# vlan VLAN-ID
```

Optional parameters:

```
Sw(config-vlan)# name VLAN-NAME
Sw(config-vlan)# mtu MTU-SIZE
!default: 1500
```

To verify:

```
Sw# show vlan [name VLAN-NAME| id VLAN-ID]
```

### Normal Range vs Extended Range

* **Normal Range VLANs**: 1-1005\* – Supported by VTP v1,v2,v3 in all modes.
* **Extended Range VLANs**: 1006-4094 – Only supported in Transparent mode by VTPv1 and v2, and in all modes by VTP v3.

\* Reserved VLANs: 1002-1005 (Token Ring and FDDI)

## Port Modes

### Static Access

```
Sw(config-if)# switchport mode access
```

#### **2.1.1Access with static VLAN assignment**

Default VLAN for all ports is VLAN 1.To change it, use:

```
Sw(config-if)# switchport access vlan VLAN-ID
```

#### **Access with dynamic VLAN assignment**

```
Sw(config-if)# switchport access vlan dynamic
```

This configuration requires a VMPS Server (VLAN Management Policy Server).

### Static Trunk

First, Define the encapsulation type:

```
Sw(config-if)# switchport trunk encapsulation {dot1q|isl|negotiate}
! dot1q =  802.1q IEEE Standard
! isl = Cisco Proprietary (prefered)
! negotiate = used if dynamic trunking is used (default)
```

To configure the port as a static Trunk port, use:

```
Sw(config-if)# switchport mode trunk
```

#### **Native VLANs**

On 802.1q trunks, frames in the native vlan are sent untagged. To set the native vlan, use:

```
Sw(config-if)# switchport trunk native vlan VLAN-ID
! Default Native VLAN: 1.
```

#### **Allowed VLANs**

By default, all VLANs are allowed on a trunk. To limit the VLANs that can travel over a trunk link, use:

```
Sw(config-if)# switchport trunk allowed vlan {add VLAN-LIST| remove VLAN-LIST| except VLAN-LIST|all}
```

### Dynamic

A port configure in dynamic mode, will become either an access or a trunk port, depending on the negotiation with the other pot it is connected to.\
DTP is used to negotiate Dynamic Trunks.\
To configure a port as dynamic, use the following command:

```
Sw(config-if)# switchport mode dynamic {auto|desirable}
!Default setting is platform dependent
```

Here’s how the ports will end up in a dynamic configuration:

| This side\The other side | Access        | Trunk        | Dynamic Auto  | Dynamic Desirable |
| ------------------------ | ------------- | ------------ | ------------- | ----------------- |
| Access                   | access\access | access\trunk | acces\access  | access\access     |
| Trunk                    | trunk\access  | trunk\trunk  | trunk\trunk   | trunk\trunk       |
| Dynamic Auto             | access\access | trunk\trunk  | access\access | trunk\trunk       |
| Dynamic Desirable        | access\access | trunk\trunk  | trunk\trunk   | trunk\trunk       |

DTP works by default, even if the port is configured as a static trunk. This is needed so that the other end of the connection could negotiate to become a trunk.\
DTP is disabled if the port is set in the acccess mode or if the following command is used:

```
Sw(config-if)# switchport nonegotiate
```

This is usually used when a static trunk is created with a neighbor that does not support DTP (like a router or a firewall).\
DTP negociation will fail if the devices are in different VTP domains.

### Private VLAN

See [Private VLANs](private-vlans.md)

### Dot1Q Tunnels (Q-in-Q Tunnels)

Q-in-Q tunnels are used to carry frames tagged with 802.1q by a customer over a provider’s network. Customer traffic is encapsulated within another 802.1q tag (metro tag) which is used inside the provider network.\
An asymmetric link must be configured, where the port on the customer switch will be set as Trunk port, and the port on the Provider switch will be set as Tunnel Port.

#### **Configure the Customer Switch**

To configure the customer switch, use:

```
SwC(config-if)# switchport mode trunk
```

#### **Configure the Provider Switch**

```
SwP(config-if)# switchport access vlan VLAN-ID
! VLAN-ID is the metro tag, specific to each customer
SwP(config-if)# switchport mode dot1q-tunnel
! Will also disable CDP on the port
```

Since Q-in-Q tunnels add an additional dot1Q header, the MTU of the frames can reach 1504 Bytes. The switch will warn when a port is configured for dot1q tunneling that the default MTU of the switch (1500) should be changed. This change should be done on all provider switches:

```
Sw(config)# system mtu 1504
! Requires a reload
```

To test that the provider network can accomodate such frames, you can try:

```
SwC# ping IP-ADDR size 1500 df-bit
```

#### **Native VLAN on Dot1q Tunnels**

Since trunk belonging to the native VLAN is normally sent untagged, this could end up in problems inside the provider network. To prevent this use one of the following:

* Use ISL encapsulation inside the provider network
* Make sure the native VLAN on the customer/provider edge is not within the customer VLAN range.
*   Tag the native VLAN:

    ```
    Sw(config)# vlan dot1q tag native
    ```

#### **Tunneling L2 Protocols**

Normally traffic for VTP, CDP, STP, PAgP, LACP, UDLD is not switched. It is interpreted by each device on a per-link basis.\
To enable L2 Protocl Tunneling, use the following config on the provider switch:

```
SwP(config-if)# l2protocol-tunnel [cdp|stp|vtp]
```

To tunnel protocols used over point-to-point connections, use the following command to emmulate such a connection:

```
SwP(config-if)# l2protocol-tunnel point-to-point {pagp|lacp|udld}
```

The number of L2 packets that can be tunneled can be limited, using:

```
SwP(config-if)# l2protocol-tunnel drop-threshold [cdp|stp|vtp|point-to-point {pagp|lacp|udld}] VALUE
! if no protocol is specified, the limit applies to all protocols
! Packets over the specified threshold will be dropped
```

You can also configure the interface to shutdown if a threshold is violated:

```
SwP(config-if)# l2protocol-tunnel shutdown-threshold [cdp|stp|vtp|point-to-point {pagp|lacp|udld}] VALUE
! if no protocol is specified, the limit applies to all protocols
```

To auto-recover from such an error, use:

```
SwP(config)# errdisable recovery cause l2ptguard
```

To monitor,use:

```
SwP#show l2protocol-tunnel [summary|interface INTERFACE]
```
