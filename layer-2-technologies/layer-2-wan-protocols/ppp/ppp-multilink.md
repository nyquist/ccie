# PPP Multilink

## PPP Mulitlink

One feature that PPP can offer is the posibility to bundle multiple physical links into one logical link, much like an EtherChannel interface available on the Ethernet Switches.\
Considering two routers connected back-to-back over more than one serial links. Each serial interface can be configured for PPP encapsulation and assigned to a Multilink Group:

```
R(config-if)# encapsulation ppp
R(config-if)# ppp multilink group GROUP
```

Then, you can use the virtual Multilink interface to assign an IP Address:

```
R(config)# interface Multilink GROUP
R(config-if)# ppp multilink
R(config-if)# ip address IP-ADDR NETMASK
```

By default the peer neighbor-route feature is eanbled, and we will see /32 routes in the routing table.

The link will be operational as long as one link is active. This behaviour can be changed using the command:

```
R1(config-if)#ppp multilink links minimum MIN [mandatory]
```

_MIN_ specifies the minimum number of links needed to be up before the multilink interface goes up. When the **mandatory** keyowrd is used, if the number of active links goes below the number of minimum links, the multilink interface goes down. If **mandatory** is not used, the minimum links value is checked only when the interface first comes up. If the number of active links drops below minimum while the interface is up, the multilink will still remain up as long as one link is operational.

You can check the status of the multilink using:

```
R1#sh ppp multilink
Multilink1, bundle name is R2
Endpoint discriminator is R2
Bundle up for 00:14:20, total bandwidth 3088, load 1/255
Receive buffer limit 24000 bytes, frag timeout 1000 ms
0/0 fragments/bytes in reassembly list
30 lost fragments, 53 reordered
16/918 discarded fragments/bytes, 0 lost received
0x89 received sequence, 0x2E sent sequence
Member links: 2 active, 0 inactive (max not set, min not set)
Se1/2, since 00:14:20
Se1/0, since 00:12:54
No inactive multilink interfaces
```

## PPP Multilink over Frame Relay

PPP Multilink over Frame Relay is configured similarly. The steps to enable it are:\
1\. Configure the Serial interfaces for Frame Relay encapsulation

```
R(config)# interface INTERFACE
R(config-if)# encapsulation frame-relay
```

2\. Configure the Serial interfaces for PPP:

```
R(config-if)# frame-relay interface-dlci DLCI ppp VIRTUAL-TEMPLATE-INTERFACE
```

3\. Configure the Virtual Template to be part of the Multilink Group

```
R(config)# interface VIRTUAL-TEMPLATE-INTERFACE
R(config-if)# ppp multilink
R(config-if)# ppp multilink group GROUP
```

4\. Configure the Multilink interface

```
R(config)# interface MULTILINK-INTERFACE
R(config-if)# ppp multilink
R(config-if)# ppp multilink group GROUP
R(config-if)# ip address...
```
