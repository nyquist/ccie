# IPv6 Routing

## Enable IPv6 Routing

To enable IPv6 routing, use:

```
R(config)# ipv6 unicast-routing
```

This will enable the router to send RAs unsolicited or in response to RS messages.

## Static Routing

```
! Directly attached static routes:
R(config)# ipv6 route IPV6-PREFIX/IPV6-PREFIX-LEN OUT-INTERFACE [AD]
! Recursive routes:
R(config)# ipv6 route IPV6-PREFIX/IPV6-PREFIX-LEN IPV6-NEXT-HOP [AD]
! Fully Specified Static routes:
R(config)# ipv6 route IPV6-PREFIX/IPV6-PREFIX-LEN INTERFACE IPV6-NEXT-HOP [AD]
```

To see the routing table, use:

```
R# show ipv6 route
```

## RIP for IPv6

RIP for IPv6, aka RIPng, works just like RIPv2 for IPv4. It sends multicast packets to FF02::9 multicast address.

### Start the process

To start the RIP process, enable RIP on each interface where it should run:

```
R(config-if)# ipv6 rip PROC-NAME enable
```

RIP uses the link-local addresses as RIP source address, so in Frame Relay you would probably need static mapping. There can be multiple instances of RIPng running on the same router and you can set each to listen on a different UDP port on the same subnet.

```
R(config)# ipv6 router rip PROC-NAME
R(config-rtr)# port PORT multicast-group MULTICAST-ADDR
```

### Default routes

To generate a default route use the command:

```
R(config-if)# ipv6 rip PROC-NAME default-information {originate|only} [metric METRIC]
! originate - advertises the default and the other RIP routes
! only - advertises only the default (like a summary)
```

## EIGRP for IPv6

### Start the process

Enable EIGRP on each interface:

```
R(config-if)# ipv6 eigrp AS-NUMBER
```

By default, the EIGRPv6 process is shutdown, so we must enable it with:

```
R(config)# ipv6 router eigrp AS-NUMBER
R(config-rtr)# no shut
```

If there are no IPv4 addresses on the router, it cannot create a Router ID and the process won’t start. You will have to statically define one:

```
R(config-rtr)# router-id IPV4-ADDRESS
```

There is no auto-summary in EIGRP for IPv6.

EIGRP uses the link-local addresses to create adjacencies so you might need to set static mappings in Frame Relay.

EIGRP for IPv6 uses FF02::A address for multicast messages.

There are no network statements in EIGRP for IPv6, only interface statements

## OSPFv3 for IPv6

It works similar to OSFPv2.

### Start the OSPFv3 Process

Assign interfaces to OSPFv3:

```
R(config-if)# ipv6 ospf PROC-ID area AREA-ID
```

If there are no IPv4 addresses on the router, it cannot create a Router ID and the process won’t start. You will have to statically define one:

```
R(config)# ipv6 router ospf PROC-ID
R(config-rtr)# router-id IPV4-ADDRESS
```

OSPF uses the link-local addresses to create adjacencies so you might need to set static mappings in Frame Relay.

### Summarization

```
! at the ABR:
R(config-rtr)# area AREA range IPV6-PREFIX
! at the ASBR
R(config-rtr)# summary-prefix IPV6-PREFIX [not-advertise|tag TAG]
! not-advertise will filter both the summary and the children
```

## MP-BGP for IPv6

IPv6 can be advertised via MP-BGP. It is just another address-family available in the normal BGP process.

### Start MP-BGP for IPv6

```
R(config)# router bgp AS-NUMBER
R(config-router)# neighbor IPV6-NEIGH-ADDR remote-as REMOTE-AS
R(config-router)# address-family ipv6
R(config-router-af)# neighbor IPV6-NEIGH-ADDR activate
```

If there are no IPv4 addresses on the router, it cannot create a Router ID and the process won’t start. You will have to statically define one:

```
R(config-router)# bgp router-id IPV4-ADDRESS
```

## Redistribution

By default with IPv4 protocols, when redistributing from one protocol to another, the connected routes, matched by the network command where also redistributed. With IPv6 routing protocols, they are not redistributed by default anymore. Instead you will have to use the **include-connected** keyword:

```
R(config-rtr)# redistribute PROTOCOL [PROCESS|AS-NUMER] [include-connected] OPTIONS
```

## IPv6 Policy Based Routing

It works similar to [IPv4 PBR](https://nyquist.eu/policy-based-routing/).\
To enable the policy, use:

```
R(config-if)# ipv6 policy route-map ROUTE-MAP
! for locally originated traffic
R(config)# ipv6 local policy ROUTE-MAP
```
