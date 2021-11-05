# ODR

## What is ODR?

**ODR**(On Demand Routing) is a routing protocol that runs on top of CDP and that is best suited for a hub and spoke topology. If the CDP traffic is already active between the routers on a **Hub and Spoke** network, the use of ODR makes sense because the connections will not have to transport routing updates for an IGP routing protocol, relying instead on the CDP packets to carry routing information.

Of course, in terms of features, ODR can’t be compared to other more known IGPs like OSPF, EIGRP or even RIP, but in a simple hub and spoke topology, it can make wonders. The key is to be able to transport CDP frames from the hub to every spoke. This is normally the case with Frame Relay or MPLS, but not whithin a Switched Ethernet environment, where CDP would only work on a hop-by-hop basis, so the router’s will see the switches as neighbors, not each other. But there are situations where ODR can work even over ethernet switches:

1. When using dumb switches – They will not intercept CDP packets, but will flood them out all ports.
2. With Cisco swithces (and not just Cisco), using a feature that supports CDP tunneling, like 802.1Q (Q-in-Q) tunneling

Configuring ODR is easy beacuase ODR is enabled ONLY on the hub router with the command:

```
R1(config)# router odr
```

The hub router will then add routing information to the CDP packets it sends to the other spokes, informing them of a default route pointing towards itself. The nice thing is that on the spokes you don’t have to do anything and the routes will be auto-magically set. What can you do to stop the ODR default route from being installed in the spoke’s routing table?

1.  You can disable CDP

    ```
    !On an interface:
    R(config-if)# no cdp enable
    ! or globally:
    R(config)# no cdp run
    ```
2. You can start another routing protocol. When this happens, the router will stop processing and sending ODR updates!
3. You can even start ODR on the spoke.\
   When this happens, the spoke will act just like the hub in terms of processing routing information. It will send default routes toward other spokes with himself as next hop but will not exchange routes with other routers configured for ODR. Of course, this will only work if your topology is more of a “partial mesh” because there are practically 2 hubs and the spokes should connect to both of them directly so they can talk over CDP. The spokes will install default routes from each router and this way they will be able to load-balance traffic between the 2 hubs.

ODR has an adminitrative distance of 160 making it less preferred compared to most other protocols. In fact, only External EIGRP routes (AD=170) and Internal BGP(AD=200) are less preferred than ODR.\
ODR supports VLSM and will advertise to the hub all directly connected networks, but only based on the primary IP addresses, not the secondary addresses. ODR will be able to redistribute routes to and from other routing protocols on the hub and it can even do route filtering with the classic command:

```
R(config-rotuer)# distribute-list ACL {in|out} INTERFACE
```

## Just an example

Let’s look now at an example of ODR used in a Frame Relay Topology

ODR Example topology

In this example we will use R1 as the Hub with R2 and R3 as the spokes. R1 and R2 will use the physical interfaces while R3 will use a point-to-point subinterface to the Frame Relay cloud. Each router will have a loopback interface and we will configure ODR on R1 to enable connectivity between each router.

#### 2.1 Basic Frame Relay configuration

```
! Basic config on R1 (HUB):
R1(config)# interface Serial1/1
R1(config-if)# ip address 123.0.0.1 255.255.255.0
R1(config-if)# encapsulation frame-relay
R1(config-if)# no shut
R1(config-if)# exit
R1(config)# interface Lo0
R1(config-if)# ip address 1.1.1.1 255.255.255.255
! Basic config on R2 (SPOKE)
R2(config)# interface Serial1/2
R2(config-if)# ip address 123.0.0.2 255.255.255.0
R2(config-if)# encapsulation frame-relay
R2(config-if)# no shut
R2(config-if)# exit
R2(config)# interface Lo0
R2(config-if)# ip address 2.2.2.2 255.255.255.255
! Basic config on R3 (SPOKE)
R3(config)# interface Serial1/3
R3(config-if)# encapsulation frame-relay
R3(config-if)# no shut
R3(config-if)# interface serial1/3.301
R3(config-subif)# ip address 123.0.0.1 255.255.255.0
R3(config-subif)# frame-relay interface-dlci 301
R3(config-if)# exit
R3(config)# interface Lo0
R3(config-if)# ip address 3.3.3.3 255.255.255.255
```

After LMI and Inverse ARP information is exchanged we end up with:

```
! On R1:
R1#sh frame-relay map
Serial1/1 (up): ip 123.0.0.2 dlci 102(0x66,0x1860), dynamic,
              broadcast,, status defined, active
Serial1/1 (up): ip 123.0.0.3 dlci 103(0x67,0x1870), dynamic,
              broadcast,, status defined, active
! On R2:
R2#sh frame-relay map
Serial1/2 (up): ip 123.0.0.1 dlci 201(0xC9,0x3090), dynamic,
              broadcast,, status defined, active
! On R3:
R3#sh frame-relay map
Serial1/3.301 (up): point-to-point dlci, dlci 301(0x12D,0x48D0), broadcast
          status defined, active
```

#### 2.2 Is CDP working?

In order to make ODR run, we should check if CDP is running:

```
! On R1
R1#sh cdp interface | i Serial
Serial1/0 is administratively down, line protocol is down
Serial1/2 is administratively down, line protocol is down
Serial1/3 is administratively down, line protocol is down
! On R2
R2#sh cdp interface | i Serial
Serial1/0 is administratively down, line protocol is down
Serial1/1 is administratively down, line protocol is down
Serial1/3 is administratively down, line protocol is down
! On R3
R3#sh cdp interface | i Serial
Serial1/0 is administratively down, line protocol is down
Serial1/1 is administratively down, line protocol is down
Serial1/2 is administratively down, line protocol is down
Serial1/3.301 is up, line protocol is up
```

Notice that interfaces Serial1/1 on R1 and Serial1/2 on R2 do not run CDP, while the subinteface Serial1/3.301 on R3 runs CDP! This is because, when using Frame Relay, CDP is enabled by default **only** on point-to-point subinterfaces. We will need to activate CDP now on both R1 and R2:

```
! On R1:
R1(config)# interface s1/1
R1(config-if)# cdp enable
! On R2
R2(config)# interface s1/2
R2(config-if)# cdp enable
```

Let’s verify CDP neighbors now:

```
! On R1:
R1#sh cdp neighbors
Capability Codes: R - Router, T - Trans Bridge, B - Source Route Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater

Device ID        Local Intrfce     Holdtme    Capability  Platform  Port ID
R2               Ser 1/1            128        R S I      3725      Ser 1/2
R3               Ser 1/1            169        R S I      3725      Ser 1/3.301
! On R2:
R2#sh cdp nei
Capability Codes: R - Router, T - Trans Bridge, B - Source Route Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater

Device ID        Local Intrfce     Holdtme    Capability  Platform  Port ID
R1               Ser 1/2            124        R S I      3725      Ser 1/1
! On R3:
R3#sh cdp nei
Capability Codes: R - Router, T - Trans Bridge, B - Source Route Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater

Device ID        Local Intrfce     Holdtme    Capability  Platform  Port ID
R1               Ser 1/3.301        166        R S I      3725      Ser 1/1
```

#### 2.3 Let’s fire up ODR

Now that we have CDP working, we should enable ODR on the hub:

```
R1(config)# router odr
```

That’s it! Let’s check the routing table now:

\[cisco highlite=”7,9,20,29″]! On R1:\
Gateway of last resort is not set

1.0.0.0/32 is subnetted, 1 subnets\
C 1.1.1.1 is directly connected, Loopback0\
2.0.0.0/32 is subnetted, 1 subnets\
o 2.2.2.2 \[160/1] via 123.0.0.2, 00:00:57, Serial1/1\
3.0.0.0/32 is subnetted, 1 subnets\
o 3.3.3.3 \[160/1] via 123.0.0.3, 00:00:16, Serial1/1\
123.0.0.0/24 is subnetted, 1 subnets\
C 123.0.0.0 is directly connected, Serial1/1\
! On R2:\
R2#sh ip route\
Gateway of last resort is 123.0.0.1 to network 0.0.0.0

2.0.0.0/32 is subnetted, 1 subnets\
C 2.2.2.2 is directly connected, Loopback0\
123.0.0.0/24 is subnetted, 1 subnets\
C 123.0.0.0 is directly connected, Serial1/2\
o\* 0.0.0.0/0 \[160/1] via 123.0.0.1, 00:00:37, Serial1/2\
! On R3:\
R3#sh ip route\
Gateway of last resort is 123.0.0.1 to network 0.0.0.0

3.0.0.0/32 is subnetted, 1 subnets\
C 3.3.3.3 is directly connected, Loopback0\
123.0.0.0/24 is subnetted, 1 subnets\
C 123.0.0.0 is directly connected, Serial1/3.301\
o\* 0.0.0.0/0 \[160/1] via 123.0.0.1, 00:00:23, Serial1/3.301

Well, it looks pretty good. We have default routes on the spokes and the hub knows how to get to all networks. Now let’s test this, just to make sure:

```
!On R1:
R1#ping 2.2.2.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 28/36/40 ms
R1#ping 3.3.3.3

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 20/40/68 ms
!On R2:
R2#ping 1.1.1.1

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 24/33/40 ms
R2#ping 3.3.3.3

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 3.3.3.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 56/77/88 ms
!On R3:
R3#ping 1.1.1.1

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 24/36/48 ms
R3#ping 2.2.2.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```

Wow! It all seemed to go well until the last ping. What happend? How come we can ping R3 from R2 but not R2 from R3?

#### 2.4 What went wrong?

Well, this doesn’t really have to do with ODR anymore. ODR did its job – we have correct routing information on all routers.\
What happens is that when we ping R3 from R2, we actually send packets with source 123.0.0.2 and destination 3.3.3.3. R2 will look in its routing table and will find that is should send packets for 3.3.3.3 using the default route. The default route points to 123.0.0.1, and we have an Inverse ARP map for that – 102. The packet is sent out to R1. On R1, it will be redirected to R3. When R3 receives the ICMP packet, in order to reply, it will create a packet with a source of 3.3.3.3 and destination of 123.0.0.2.

**Now here’s why R2 can ping R3’s loopback**: To send to 123.0.0.2 the routing table points to the directly connected interface Serial1/3.301. This is a point-to-point interface and R3 doesn’t care about Inverse ARP or static maps on this interface. It will just send the packet using the DLCI assigned to it. THe packet arrives at R3 who redirects it to R2 and we have connectivity!

In the same way, when we ping R2 from R3, we actually send packets with source 123.0.0.3 and destination 2.2.2.2. R3 looks up a route to 2.2.2.2 It doesn’t find one so it uses the default route that points to 123.0.0.1 on the point-to-point Serial1/3.301. R3 just encapsulates the packet using DLCI 301 and sends it out. R1 receives the packet and redirects it over to R2.

**Now here’s why R3 can’t ping R2’s loopback**: R2 receive the packet and must send a reply from source ip 2.2.2.2 to destination IP 123.0.0.3. It looks up the destination in the routing table, finds out it’s Serial1/2 (directly connected) but we don’t have a mapping for 123.0.0.3 and we don’t know what DLCI to use to reach this IP. Encapsulation fails, the ping fails.

#### 2.5 Solutions?

The solution? We can just ping R2’s loopback using R3’s loopback. There’s no problem with this

```
R3#ping 2.2.2.2 source 3.3.3.3

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
Packet sent with a source address of 3.3.3.3
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 64/76/92 ms
```

Or, we can add a static map on R2 that tells the router to use the same DLCI 201 to reach 123.0.0.3, also.

```
! On R2:
R2(config)#int s1/2
R2(config-if)#frame-relay map ip 123.0.0.3 201
! On R3:
R3#ping 2.2.2.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 64/81/92 ms

3. Conclusions
ODR works great, but there's a difference between having a route and having connectivity. Especially on Frame Relay.
							
					
```
