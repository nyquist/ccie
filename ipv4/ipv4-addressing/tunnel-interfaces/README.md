# Tunnel Interfaces

## Tunnel Modes

A tunnel makes two distant devices appear directly connected over a logical interface. When a packet is sent out on the tunnel interface, it is encapsulated in the “carrier” protocol and sent over a physical interface.The most used carrier protocols are GRE, IP-in-IP and IPv6, and this can be set using the tunnel mode.

When configuring a tunnel you must set a tunnel source, a destination and the carrier protocol:

```
R(config)# interface TUNNEL
R(config-if)# tunnel source {SRC-ADDR|SRC-INTERFACE}
R(config-if)# tunnel destination DEST-ADDR
R(config-if)# tunnel mode MODE
```

The tunnel source can be defined as either the Layer3 address or as an interface, but the destination can only be an address on the remote device.\
The tunnel mode specifies the carrier protocol. For IPv4, the most commonly used methods of tunneling are GRE and IPIP.

```
R(config-if)# tunnel mode {gre ip|ipip}
! gre ip - Encapsulates IPv4 in GRE
! ipip - Encapsulates IPv4 in IPv4
```

After this, you can define the encapsulating protocol’s address on the tunnel interface:

```
R(config-if)# ip address {unnumbered INTERFACE| IP-ADDR NETMSK}
```

When you have isolated networks running IPv6, you can connect them over an IPv4 backbone using tunnels. Common options are:

```
R(config-if)# tunnel mode {gre ipv6|ipv6ip [6to4|auto-tunnel|isatap]}
! gre ipv6 - Encapsulates IPv6 in GRE
! ipv6ip - Encapsulates IPv6 in IPv4
```

More details about configuring IPv6 tunnels can be found [here](https://nyquist.eu/interconnecting-ipv6-and-ipv4/#2\_Tunnels)

Another option is to have ipv6 as the transport protocol:

```
R(config-if)# tunnel mode ipv6
```

The transported protocols can be IPv4 or IPv6, based on the type of address defined on the tunnel interface.

### GRE (Generic Routing Encapsulation)

GRE is defined as IP Protcol 47. It adds a 20 byte IP header and 4 byte GRE header to an existing packet so that it can be routed based on this new information.

#### GRE Keepalives

GRE supports sending and monitoring keepalives to determine the status of a tunnel interface.

```
R(config-if)# keepalive [PERIOD [RETRIES]]
! Default PERIOD: 10 sec
! Default RETRIES: 5
```

## Path MTU Discovery

Tunneling packets means an extra encapsulation header that is added to the packet which can make the packet too big on some links.\
To set the MTU value on a tunnel you can set it manually or use auto-discovery:

```
R(config-if)# ip mtu MTU
R(config-if)# tunnel path-mtu-discovery [age-timer {TIME|infinte}| min-mtu SIZE]
! Default TIME: 10 min
```

The discovery will run for a limited ammount of time unless the infinte keyword is used. Using SIZE, you can configure a minimum size of the discovered MTU that can be accepted. If the timer expires, the router will choose a MTU equal to the default interface MTU-20 Bytes for IP-in-IP or default interface MTU-24 Bytes for GRE.\
Path MTU discovery is only available on GRE and IPIP tunnels.

## VRF Support

The transport protocol of one tunnel can run in one VRF, while the transported protocol can run in another VRF. By default, both the transport and the transported protocols run in the global VRF. You can change the VRF of the transporting protocol with:

```
R(config-if)# tunnel vrf VRF
```

Source and destination addresses of the tunnel must run in this VRF.\
To change the VRF of the transported protocol, use:

```
R(config-if)# ip vrf forwarding vRF
```

The addresses defined on the tunnel will run in this VRF.
