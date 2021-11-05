# Policy based Routing

## In Theory

Policy Routing is a feature on Cisco routers that offers greater control over paths in a network.\
Normal routing is done based on the routing table. The router looks up the destination address of a packet and finds out the next hop or the outgoing interface.\
With Policy Routing you can override whatever the routing table says and just route packets however you like.\
Isn’t this what static routing does? To some degree, yes. You can use static routes to manipulate the outcome of a route lookup, but static routes integrate in the normal routing process. The router will still base it’s routing decision on the packet’s destination address.\
The true power of policy routing is that it can take routing decisions on other parameters like incoming interface, source address or even application type (http, ftp).

Configuring Polcy Routing is a 2 step process.

### Define the route-map

```
R(config)# route-map ROUTE-MAP [permit|deny] [SEQ]
!default: permit 10
```

A route-map can have multiple statements, each statement being identified by a **SEQ**. If you don’t set a number, the default is 10. One statement can have multiple match commands and multiple set commands.

A packet is passed sequentially through the route-map statements. At each statement, the packet is considered matched if it is matched by all the match commands in that statement. If at least one of the match statements is not matched, then the packet is passed to the next statement. If all match statements are matched, then all set commands are applied.

If a matched route-map statement is set to permit, then the packet is policy-routed. If a matched route-map statement is set to deny, then it is routed according to the routing table. In either case, once a matching statement is found, no other statements are checked. If no statements are matched the packet is also routed according to the routing table.

Inside the route-map configuration mode we have to define a matching criteria using the **match** command. There are a lot of options, but most of them have to do with manipulating routing updates. For the purpose of Policy Routing we can use the following commands. In one statement, there can be multiple match commands, and a packet will be considered to match if it matches all of them (AND logic). One match command can refer to multiple objects (ex: multiple ACLs). A packet will be considered to match that entry if it matches at least one of the objects (OR logic).

```
R(config-route-map)# match interface INTERFACE
!Matches on incoming interface:
R(config-route-map)# match ip address ACL
!Matches on destination IP:
R(config-route-map)# match ipv6 address ACL
!Matches on IPv6 destination IP:
R(config-route-map)# match length MIN-LEN
!Matchs on minimum packet length:
```

When a packet is matched, all the **set** commands that are configure in the statement are executed:

```
R(config-route-map)# set [default] interface INTERFACE
! Sets the outgoing interface
R(config-route-map)# set ip [default] next-hop NEXT-HOP-IP [recursive NEXT-HOP-IP2]
! Sets the next-hop address (IPv4)
R(config-route-map)# set ipv6 [default] next-hop NEXT-HOP-IP
! Sets the next-hop address (IPv6)
```

When using the **default** keyword we tell the router to first try to find a match using the routing table and then use our defined policy, basically overriding only the routing table’s default route.

As you can see, we can define an outgoing interface or a next-hop address. Just like with static routes, you must understand the differences between using the next-hop or the outgoing interface. If there are multiple set commands, the order in which they are resolved is:

1. set ip next-hop
2. set ip next-hop recursive
3. set interface
4. set ip default next-hop
5. set ip default interface

For IPv4 we have one more option – using EOT – Enhanced Object Tracking to see if the next hop is up or not:

```
R(config-route-map)#set ip next-hop verify-availability NEXT-HOP-IP SEQ track OBJ
```

**SEQ** is a sequence number that is used to identify multiple options for the next hop, end **OBJ** is the object used to track availability. As you can imagine, this next hop will be used only if the object used for tracking is returning a positive result.

Optionally, you can also set a description for the route-map entry

```
R(config-route-map)# description STRING
```

#### **Use route maps for traffic filtering**

Simply match incoming traffic and set the outgoing interface to NULL0:

```
R(config-route-map)#set interface NULL0
```

### Apply the Route Map

You probably realized that the policy must be set on the interface where we expect to receive the traffic that will be matched. Setting the route-map on the outgoing interface makes no sense because the packets have already went through the routing process so they will just be sent out.

```
R(config-if)# ip policy route-map ROUTE-MAP
```

But what if I want to policy route traffic originated from the router. It has no incoming interface, so we have to use:

```
R(config)# ip local policy route-map ROUTE-MAP
```

#### **Force local traffic to be checked by ACLs**

By default, traffic originated locally is not checked by ACLs set on interfaces. You can use route-maps to set the next-hop of locally originated traffic to a loopback interface, and force this way the traffic from being checked by the ACLs. The loopback interface is still local but the fact that is switching from one interface to another will force the router to apply the ACLs on this traffic.

## Working examples

### Example 1

Here’s an example:

[![](https://nyquist.eu/wp-content/uploads/2012/03/PBR.png)](https://nyquist.eu/wp-content/uploads/2012/03/PBR.png)

PBR Example Topology

In our topology, we have 5 routers, each configured with an ip address on the Lo0 interface. On R1 we will simulate 2 hosts by using Lo1, as well. We will run EIGRP for route exchange. Here’s the starting config:\
On R1:

```
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
!
interface Loopback1
 ip address 11.11.11.11 255.255.255.255
!
interface FastEthernet0/0
 ip address 12.0.0.1 255.255.255.0
 duplex auto
 speed auto
!
router eigrp 100
 network 0.0.0.0
 no auto-summary
```

On R2:

```
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
!
interface FastEthernet0/0
 ip address 12.0.0.2 255.255.255.0
 duplex auto
 speed auto
!
interface FastEthernet0/1
 ip address 24.0.0.2 255.255.255.0
 duplex auto
 speed auto
!
interface Serial1/0
 ip address 23.0.0.2 255.255.255.0
 serial restart-delay 0
!
router eigrp 100
 network 0.0.0.0
 no auto-summary
```

On R3:

```
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
!
interface FastEthernet0/1
 ip address 35.0.0.3 255.255.255.0
 duplex auto
 speed auto
!
interface Serial1/0
 ip address 23.0.0.3 255.255.255.0
 serial restart-delay 0
!
router eigrp 100
 network 0.0.0.0
 no auto-summary
```

On R4:

```
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
!
interface FastEthernet0/0
 ip address 45.0.0.4 255.255.255.0
 duplex auto
 speed auto
!
interface FastEthernet0/1
 ip address 24.0.0.4 255.255.255.0
 duplex auto
 speed auto
!
router eigrp 100
 network 0.0.0.0
 no auto-summary
```

On R5:

```
interface Loopback0
 ip address 5.5.5.5 255.255.255.255
!
interface FastEthernet0/0
 ip address 45.0.0.5 255.255.255.0
 duplex auto
 speed auto
!
interface FastEthernet0/1
 ip address 35.0.0.5 255.255.255.0
 duplex auto
 speed auto
!
router eigrp 100
 network 0.0.0.0
 no auto-summary
```

Remember that by default, physical router interfaces are disabled, so you should enable them using

```
R(config-if)# no shutdown
```

On R1 we can test connectivity to R5 and see how the network converged:

```
R1#traceroute 5.5.5.5 source 1.1.1.1 numeric

Type escape sequence to abort.
Tracing the route to 5.5.5.5

  1 12.0.0.2 8 msec 24 msec 16 msec
  2 24.0.0.4 28 msec 36 msec 48 msec
  3 45.0.0.5 44 msec *  88 msec

R1#traceroute 5.5.5.5 source 11.11.11.11 numeric

Type escape sequence to abort.
Tracing the route to 5.5.5.5

  1 12.0.0.2 12 msec 24 msec 16 msec
  2 24.0.0.4 36 msec 36 msec 40 msec
  3 45.0.0.5 72 msec *  56 msec
```

Use **numeric** when starting a traceroute to disable the DNS lookups and get a faster response. As you can see, the route goes through R2, then R4 and then R5. The reason for this is because on R2, the interface towards R4 is Fast Ethernet, while the link towards R3 is over a HDLC serial link. Since the default EIGRP metric involves bandwidth and delay the route towards R4 will be considered the best. If we had RIP running, it would have installed 2 equal routes towards R5, as the hop count is the same.

```
R2#sh int fa0/1 | i BW
  MTU 1500 bytes, BW 10000 Kbit, DLY 1000 usec,
R2#sh int s1/0 | i BW
  MTU 1500 bytes, BW 1544 Kbit, DLY 20000 usec,
```

Now let’s set a route map on R2 so that traffic from 11.11.11.11 would go through R3:

```
!Define an ACL to match the traffic:
R2(config)# access-list 99 permit 11.11.11.11
!Define the route map
R2(config)# route-map ROUTE_MAP11 permit 10
R2(config-route-map)# match ip address 99
R2(config-route-map)# set ip next-hop 23.0.0.3
R2(config-route-map)# exit
!Apply the policy on the incoming interface:
R2(config)# interface Fa0/0
R2(config-if)# ip policy route-map ROUTE_MAP11
```

To verify the configuration, we will traceroute from R1 using each Loopbacks as the source:

```
R1#traceroute 5.5.5.5 source 1.1.1.1 numeric

Type escape sequence to abort.
Tracing the route to 5.5.5.5

  1 12.0.0.2 12 msec 16 msec 20 msec
  2 24.0.0.4 40 msec 40 msec 44 msec
  3 45.0.0.5 52 msec *  68 msec

R1#traceroute 5.5.5.5 source 11.11.11.11 numeric

Type escape sequence to abort.
Tracing the route to 5.5.5.5

  1 12.0.0.2 12 msec 20 msec 20 msec
  2 23.0.0.3 48 msec 64 msec 72 msec
  3 35.0.0.5 60 msec *  72 msec
```

As you can see, when the source is 1.1.1.1, the path is R1-R2-R4-R5and when the source is 11.11.11.11, the path is R1-R2-R3-R5\
To verify on R2, we can start a debug process to see how the traffic was routed.

```
R2#debug ip policy
Policy routing debugging is on
R2#
*Mar  1 02:44:30.919: IP: s=1.1.1.1 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy rejected(no match) - normal forwarding
*Mar  1 02:44:30.955: IP: s=1.1.1.1 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy rejected(no match) - normal forwarding
*Mar  1 02:44:30.999: IP: s=1.1.1.1 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy rejected(no match) - normal forwarding
*Mar  1 02:44:31.043: IP: s=1.1.1.1 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy rejected(no match) - normal forwarding
*Mar  1 02:44:31.095: IP: s=1.1.1.1 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy rejected(no match) - normal forwarding
*Mar  1 02:44:34.095: IP: s=1.1.1.1 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy rejected(no match) - normal forwarding
*Mar  1 02:44:44.463: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy match
*Mar  1 02:44:44.463: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, g=23.0.0.3, len 28, FIB policy routed
*Mar  1 02:44:44.523: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy match
*Mar  1 02:44:44.523: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, g=23.0.0.3, len 28, FIB policy routed
*Mar  1 02:44:44.583: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy match
*Mar  1 02:44:44.587: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, g=23.0.0.3, len 28, FIB policy routed
*Mar  1 02:44:44.647: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy match
*Mar  1 02:44:44.647: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, g=23.0.0.3, len 28, FIB policy routed
*Mar  1 02:44:44.707: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy match
*Mar  1 02:44:44.707: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, g=23.0.0.3, len 28, FIB policy routed
*Mar  1 02:44:47.679: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, len 28, FIB policy match
*Mar  1 02:44:47.679: IP: s=11.11.11.11 (FastEthernet0/0), d=5.5.5.5, g=23.0.0.3, len 28, FIB policy routed
```

Or we can use this command to see the policies defined and how many packets were matched:

```
R2#sh route-map
route-map ROUTE_MAP11, permit, sequence 10
  Match clauses:
    ip address (access-lists): 99
  Set clauses:
    ip next-hop 23.0.0.3
  Policy routing matches: 6 packets, 360 bytes
```

Notice there are 6 packets that matched, because although we sent 12 packets (check the debug output), only 6 of them matched the criteria – those sourced by 11.11.11.11. Why do we have 6 packets for just one traceroute command?\
Traceroute uses the TTL field to discover the path to a destination. It first sends packets with TTL=1, then increases the TTL until it reaches the destination. Each router on the path should decrease the TTL when routing the packet and should drop it when the TTL reaches 0. Actually, if the packet is not for itself, it would be pointless to route it, then decrease TTL and drop it, So, incoming packet with TTL 1 that should be routed are dropped. When dropping, it also should send an ICMP reply with the Type 11 (time-exceeded) and Code 0 (time to live exceeded in transit) back to the source, thus helping traceroute to map each router along the path.\
Cisco routers send by default 3 UDP packets for each TTL value starting from 1 upwards. This will help find multiple routes to a destination and also compute some average values for the Round Trip Time to each router along the path.\
So R1 first sends 3 packets with TTL=1. They will be droped by R2, so they won’t even get to the routing part – Matches so far: 0\
R1 then sends another 3 packets with TTL=2. They will pass through R2 before being dropped by R3 – Matches so far: 3\
R1 then sends another 3 packets with TTL=3. They will pass through R2 and R3 before R4 responds with an ICMP Type 3 (Destination unreachable) Code 3 (Port unreachable) for each one, signaling that the packets reached the destination, but there is no one waiting this kind of traffic. Matches so far: 6\
So there it is, the 6 matches that we saw, made sense.

### Example 2

Let’s say I want to have an out-of-band management connection from R2 to R5, when connecting via telnet. Let’s enable telnet on R5 first:

```
R5(config)# line vty 0 4
R5(config-line)# password cisco
R5(config-line)# login
```

We should be able to telnet from R2 to R5, but we are going through R4. I am using 2.2.2.2 as the source address of my telnet traffic.

```
R2#telnet 5.5.5.5 /source-interface lo0
```

Let’s set this management traffic to be out-of-band, through R3:

```
! Create the ACL:
R2(config)# access-list 199 permit tcp any any eq telnet
!Create the route-map
R2(config)# route-map ROUTE_MAP_TELNET permit 10
R2(config-route-map)# match ip address 199
R2(config-route-map)# set ip next-hop 23.0.0.3
! Apply the route-map
R2(config)# ip local policy route-map ROUTE_MAP_TELNET
```

In order to verify, let’s create an ACL on R5 just for accounting purpose:

```
! First allow telnet packets for accounting
R5(config)# access-list 100 permit tcp any any eq telnet log-input
! Then allow anything else
R5(config)# access-list 100 permit ip any any
! Apply it on Fa0/1 interface
R5(config)# interface Fa0/1
R5(config-if)# ip access-group 100 in
```

Back on R2, we should be able to telnet to R5, but remember to disable debugging first, otherwise we would see a lot of messages like this:

```
*Mar  1 06:11:53.994: IP: route map ROUTE_MAP_TELNET, item 10, permit
*Mar  1 06:11:53.994: IP: s=2.2.2.2 (local), d=5.5.5.5 (Serial1/0), len 40, policy routed
```

It’s just the **debug ip policy** that was previously started, telling us that there are locally generated packets that match the route-map.\
Over to R5 to verify:

```
R5#sh access-lists 100
Extended IP access list 100
    10 permit tcp any any eq telnet log-input (57 matches)
    20 permit ip any any (265 matches)
```

Yes! The telnet traffic originated by R2 arrived on Fa0/1 via R3.

### The end?

We were able to send traffic from R2 to R5, via R3, using the route-map. But is it over? Well, no, because this is just one way. If we look at R5’s routing table, it will send return traffic back to R2’s Lo0 via R4:

```
R5#sh ip route 2.2.2.2
Routing entry for 2.2.2.2/32
Known via "eigrp 100", distance 90, metric 435200, type internal
Redistributing via eigrp 100
Last update from 45.0.0.4 on FastEthernet0/0, 01:06:22 ago
Routing Descriptor Blocks:
* 45.0.0.4, from 45.0.0.4, 01:06:22 ago, via FastEthernet0/0
Route metric is 435200, traffic share count is 1
Total delay is 7000 microseconds, minimum bandwidth is 10000 Kbit
Reliability 255/255, minimum MTU 1500 bytes
Loading 1/255, Hops 2
```

The solution is to enable PBR in the other direction also. You should know how to do it by now.
