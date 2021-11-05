# IGMP 101

To make multicast work, we use 2 types of protocols: one that is used to signal the receivers to the routers, and the other used by the routers to build distribution trees from the source to the receivers. In the first category we have IGMP (Internet Group Management Protocol), while in the second, PIM (Protocol Independent Multicast), DVMRP (Distance Vector Multicast Routing Protocol) or MSDP (Multicast Source Discovery Protocol).

## Versions

IGMP is used between hosts on a LAN and the routers to track the multicast receivers on that LAN. On Cisco routers, IGMP is enabled when PIM is enabled on the interface.

```
R# show ip igmp interface
```

There are 3 version of IGMP, and by default, Cisco IOS runs IGMPv2

```
R(config)# ip igmp version {1|2|3}
!Default: 2
```

Version 1 is defined in [RFC 1112](https://www.ietf.org/rfc/rfc1112.txt), version 2 in [RFC 2236](https://www.ietf.org/rfc/rfc2236.txt), version 3 in [RFC 3376](https://www.ietf.org/rfc/rfc3376.txt).

## How hosts JOIN and LEAVE a group

### IGMPv1

On a LAN where there are more than one routers, only one of them must act as the IGMP Querier and talk to the hosts on the network. In version 1, IGMP relies on PIM to select a Querier – the PIM DR will act as the Querier (highest IP). This differs from IGMPv2, where there is specific mechanism to elect the IGMP Querier (lowest IP).

#### **Joining a Group**

The IGMP Querier sends an IGMPv1 Membership QUERY to 224.0.0.1 (All Hosts). Each host that receives the Query stars a random timer (max 10 sec) and when the first one expires, that host sends an IGMP MEMBERSHIP REPORT to the Group Address that it wishes to join. Each host in the group receives this REPORT and will suppress it’s own report because the router does’n need to know who exactly needs the traffic, just if there is anyone who needs it.

```
R(config)# ip igmp query-interval INTERVAL
!how often will the router send IGMP QUERIES on the LAN. Default: 1 min
```

A host can also send unsolicited MEMBERSHIP REPORTS when they want to join a group.

#### **Leaving a Group**

In IGMPv1, leaving a group is only achieved via timeout. The Timeout is 3x Query Time, that is 3 minutes by default.

### IGMP v2

The IGMP Querier is elected in version 2 as the router with the lowest IP address. The time to wait before another router can become the acting querier can be defined with:

```
R(config)# ip igmp querier-timeout TIMEOUT
!Default: 255 sec
!how long before detecting that the acting Querier is gone
```

When IGMPv2 hosts detect IGMPv1 routers, they must fallback to using IGMPv1 and stop sending IGMPv2 messages until no IGMPv1 messages are received. When IGMPv2 routers detect IGMPv1 hosts, they will ignore GROUP-SPECIFIC-QUERIES and LEAVE messages because these are not understood by IGMPv1 hosts.

#### **Joining a Group**

The IGMP Querier sends 2 types of  IGMP Membership QUERIES in version 2. There is the GENERAL-QUERY that acts just like a v1 QUERY, but there is also a GROUP-SPECIFIC-QUERY, used to find if a specific group still has members.

Additionally, IGMPv2 GROUP-SPECIFIC-QUERIES offer the possibility to set the MAX-RESPONSE-TIME, the random timer used by the hosts before responding.

```
R(config)# ip igmp query-max-response-time MAX-TIME
!Default MAX-TIME: 1 sec => Quicker than GENERAL QUERY
```

#### **Leaving a Group**

IGMPv2 adds a LEAVE group message that is usually sent by any host when it leaves a group. When the router receives a LEAVE message, it sends a GROUP SPECIFIC QUERY to see if there are any receivers left on the LAN. If there are hosts, they will respond with a REPORT. If there is no response, the MAX REPONSE TIME will timeout (1 sec) and the router will send a new Group specific QUERY (default: 2 times). If again no response in time, the router will not forward traffic for this group on that segment. To change the defaults:

```
R(config-if)# ip igmp last-member-query-count COUNT
!Default: 2
R(config-if)# ip igmp last-member-query-interval INTERVAL
!Default: 1 second
```

You can also set the router to immediately prune the traffic. The receivers must then send JOIN messages if they still want to receive the traffic:

```
R(config)# ip igmp immediate-leave
```

### IGMP v3

#### **Joining a Group**

In IGMPv1 and IGMPv2, receivers would only join a destination group, and traffic from any source sent to this destination was going to reach the receivers. Version 3 introduces support for Source Specific Multicast (SSM), which means that receivers will only get the traffic sent by a specific source. This means that when receiving IGMPv3 JOINS, a router can create specific (S,G) entries in the multicast routing table (mroute). With IGMPv1 and v2, a router can only create (\*,G) entries when it receives IGMP JOINS.\
When joining an IGMPv3 group, a host can also specify the sources it wants to receive traffic from (or those that it doesn’t want to receive from). The process is similar to IGMPv2.

#### **Leaving a Group**

Leaving a group in IGMPv3 is also similar to IGMPv2, but IGMPv3 can also enable explicit tracking of each host on the network, thus quickly pruning unneeded traffic from the LAN. To enable explicit tracking, use:

```
R(config-if)# ip igmp explicit-tracking
```

#### **SSM Mapping**

Hosts must support IGMPv3 to be able to participate in SSM. But in some situations you can’t upgrade the clients to support IGMPv3 and you are stuck with them in older versions. Cisco Routers can be used to convert IGMPv1 and IGMPv2 JOINs into SSM/IGMPv3 JOINs. To do this, a router must be configured with a map of available sources for each group that the hosts will join.

```
R(config-if)# ip igmp ssm-map enable
! Enables SSM mapping
R(config-if)#  ip igmp ssm-map static GROUP-ACL SOURCE
! Enables static mapping
R(config-if)#  ip igmp ssm-map query dns
! Enables automatic mapping via DNS.
```

## Router acting as an IGMP Host

A Cisco Router can act as a host by joining a group. Since the traffic is destined to the router, it will be process switched.

```
R(config-if)# ip igmp join-group GROUP
! IGMP v1,v2,v3
R(config-if)# ip igmp join-group GROUP source SOURCE
! only IGMPv3
```

A Cisco Router can also define an interface as a static member of a group, regardless of whether there are receivers or not on that segment. The interface will appear in the OIL of the group and traffic will be fast-switched:

```
R(config-if)# ip igmp static-group {*|GROUP}
! * = all groups
R(config-if)# ip igmp static-group GROUP source {SOURCE|ssm-map}
! Only IGMPv3
```

## Filtering and limiting IGMP Joins

A router can filter incoming IGMP joins and only allow some of them, using:

```
R(config-if)# ip igmp access-group ACL
! For IGMPv1 and v2, IGMP can only be standard
! For IGMPv3, it can also be extended, matching on both group and source
```

You can also limit the number of entries in the mroute table generated by IGMP JOINs. You can do this globally:

```
R(config)# ip igmp limit MAX
```

or on an interface

```
R(config-if)# ip igmp limit MAX [except ACL]
! groups matched by the ACL will not be counted
```

The first limit that is reached is applied.

## Stub multicast routing

The multicast stub router feature is used when you can’t use PIM on a link but you still need to receive multicast traffic for a client that is connected to you. The Stub router will forward IGMP REPORTs to an upstream router which will add this interface into its OIL. To configure the stub router, use:

```
SR(config-if)# ip igmp helper-address IP-ADDR
! configure this on the stub (downstream) router
```

The upstream router should be configured to filter all PIM messages from the downstream Stub Router using one of the following commands:

```
UR(config-if)# ip pim neighbor-filter ACL
! the ACL should deny the Stub router
UR(config-if)# ip pim passive
```
