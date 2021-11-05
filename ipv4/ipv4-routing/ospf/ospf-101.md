# OSPF 101

## Starting the routing process

OSPF can be configured inside the routing process or on the interface. Even a combination of those will be a valid option. The interface configuration will override the routing process configuration.

### Inside the routing process

```
R(config)# router ospf PROCESS
R(config-router)# network NETWORK-ADDR WILDCARD area AREA
```

Depending on the IOS version, when 2 network definitions overlap, the most specific or the last one entered will override the other settings.

### On the interface

```
R(config)# interface INTERFACE
R(config-if)# ip ospf PROCESS area AREA
```

To see what interfaces run OSPF, use:

```
R#sh ip ospf interface brief
Interface    PID   Area            IP Address/Mask    Cost  State Nbrs F/C
Fa0/0        345   0               3.45.0.4/24        10    DR    1/1
Lo0          345   4               4.4.4.4/32         1     LOOP  0/0
Fa0/1        345   4               14.0.0.4/24        10    WAIT  0/0
```

OSPF will advertise a secondary subnet if it is running on the primary subnet. OSPF treats secondary subnets as stub networks so it will not send Hellos on them. Consequently, it will not form adjacencies.

### Split Horizon

OSPF does not use Split Horizon.

### Passive interfaces

When defining a passive interface, the OSPF process will advertise the network but will not send or accept OSPF messages on the interface

```
R(config-router)# passive-interface INTERFACE
```

Or, you can enable passive interfaces by default and then disable it on the interfaces where OSPF should run:

```
R(config-router)# passive-interface default
R(config-router)# no passive-interface INTERFACE
```

## Neighbors

OSPF routers send Hello packets out all OSPF enabled interfaces. When two routers receive each other’s Hellos they can become neighbors and form adjacencies. When the adjacency is up, each router will send its LSAs to the other neighbor. Each router receiving LSAs from a neighbor records the LSA in its LSDB and sends a copy of it to all of its neighbors (LSA Flooding). When the database is complete, each router uses the SPF algorithm to calculate a Loop-Free graph with itself as the root.

### Static Neighbors

Static neighbors can be defined using:

```
R(config-router)# neighbor NEIGH-ADDR [{priority PRI} | {poll-inteval DEAD-TIMER}] 
```

Defining static neighbors is a MUST on non-broadcast and point-to-multipoint non-broadcast networks. Only one side of the connection must be set with the neighbor command. The router that receives unicast hellos will respond with unicast messages to that neighbor.

### Adjacencies

When a router receives a Hello, it will check a list of parameters from the Hello packet against the values configured on the receiving interface. If they do not match, the two routers will not form an adjacency.

* **Area ID**
* **Authentication**
* **Network Mask** (except Point-to-point and virtual link interfaces)
* **HELLO interval, DEAD interval**
*   **MTU size** – It can be ignored using:

    ```
    R(config-if)# ip ospf mtu-ignore
    ```
* **Stub flag**
* **Options**

If the values match, the router checks the receiving interface Neighbor Table. If the Router ID that sent the HELLO is in the table, the DEAD interval is reset (meaning – “I already saw this neighbor”). If not, it adds the Router ID that sent the Hello to the table.

Each Hello packet contains a list of Router IDs of the originating router’s neighbors. If the receiving router finds its own Router ID in this list, it knows two-way communication is up and an adjacency can be established.

### Neighbor States

* **Down** – No HELLOs have been heard from this neighbor in the last DEAD interval. HELLOs are not sent to Down neighbors unless they are on NBMA networks (in this case Hellos are sent every Poll interval – default:120sec)
* **Attempt** – Applies only to neighbors in NBMA network, where neighbors are manually configured. A DR-eligible router transitions the state of a neighbor to Attempt when the interface to the neighbor first becomes active or when the router is the DR or the BDR. HELLOs are sent to a neighbor in the Attempt state at HELLO interval instead of POLL interval.
* **Init** – Indicates that a HELLO packet has been seen from the neighbor in the last DEAD interval, but 2-way communication has not yet been established. Starting with this state, the Router ID of the neighbor appears in the Neighbor List field of the HELLOs
* **2-Way** – This state indicates that the router has seen its own Router ID in the Neighbor field of the neighbor’s Hello packet. On multi-access networks, neighbors must be in this state or higher to be DR/BDR-eligibile. Also, receiving a Database Description packet from a neighbor in the INIT state transitions it to the 2-way state. DROthers will remain in 2-Way state with the other DROthers, and will only get to Full state with the DR and BDR.
* **ExStart** – The router and its neighbor establish a master/slave relationship and determine the initial Database Description sequence number. The neighbor with the highest Router ID becomes the master
* **Exchange** – The router sends Database Description packets to describe its entire LSD to neighbors that are in this state. The router may send LSRs to routers in this state, requesting for updated LSAs
* **Loading** – The router sends LSRs to neighbors that are in the Loading state, requesting more recent LSAs that have been discovered in the Exchange state but have not been received yet
* **Full** – The neighbors are fully adjacent and the adjacencies appear in the Router LSA and Network LSA

### Authentication

OSPF supports 3 types of authentication, but actually one of them is null authentication, which means no authentication. The 3 types of authentcation are: Type 0 (Null authentication), Type 1 (Plain Text) and Type 2 (MD5). Null authentication is used by default. To define type 1 or type 2 authentication, use the following commands:

1.  Define the authentication type, per area

    ```
    !Per Area:
    R(config-router)# area AREA authentication [message-digest]
    ! <cr> = Type 1 - Plain Text Authentication
    ! message-digest = Type 2 - MD5 Authentication
    ```

    Or per interface:

    ```
    R(config-if)# ip ospf authentication [message-digest|null]
    ! <cr> = Type 1 - Plain Text Authentication
    ! null = Type 0 - No authentication
    ! message-digest = Type 2 - MD5 Authentication
    ```
2.  Define the key to be used on the interface:

    ```
    !For plain text:
    R(config-if)# ip ospf authentication-key KEY-STRING
    !For MD5:
    R(config-if)# ip ospf message-digest-key KEY-ID md5 KEY-STIRNG
    !For MD5 both the KEY-ID and the KEY must match
    ```

Both the KEY-ID and the KEY-STRING must match for the routers to form an adjacency. The router will use the last entered key (youngest), but uses an algorithm where it sends multiple copies of the packets, each authenticated with a different key, until all neighbors use the same key. Then it uses the youngest key again.

## Timers

### Hello interval

Hello packets are sent every Hello interval

```
R(config-router)# ip ospf hello-interval HELLO
!10 sec on broadcast networks
!30 sec on non-broadcast networks
```

### Dead interval

If a router does not receive HELLOs from a neighbor in the DEAD interval it will consider the neighbor down:

```
R(config-router)# ip ospf dead-interval DEAD
!Default = 4x HelloInterval
```

When changing the HELLO time, the DEAD interval will be auto-set to 4xHello interval.

#### **Sub second convergence (Fast Hellos)**

Sets the Dead interval to 1 sec and set how many Hellos are sent in that interval

```
R(config-if)#ip ospf dead-interval minimal hello-multiplier MULTIPLIER
```

### Pacing

Pacing is a feature that groups similar packets in a single packet if they have to be sent in a short period of time.\
When LSA flooding occurs, OSPF will wait group toghether LSAs and will send them once every FLOOD-PACING interval:

```
R(config-router)# timers pacing flood MSEC
! Default: 33 msec
```

Evenry LSAs must be refreshed, checksumed or agedout. To avoid a synchronization issue, a router will group together and send packets for any of the previously mentioned operations only once every LSA-GROUP-PACING interval, instead of sending a packet for each LSA.

```
R(config-router)# timers pacing lsa-group SEC
! Default: 240 sec
```

Multiple retransmissions can be grouped into a single packet every RETRANSMISSION-PACING interval

```
R(config-router)# timers pacing retransmission MSEC
! Default: 66 msec
```

### Throttling

Throttling is a feature that rate-limits the OSPF packets.\
When a LSA is first generated, it is sent immediately. The next one is sent after MIN mseconds, and then, for each LSA that is sent, the INCREMENT timer is added. But the total timer between LSAs cannot be more than the MAX timer.

```
R(config-router)# timers throttle lsa all MIN INCREMENT MAX
! Default: MIN = 0 msec, INCREMENT = 5000 msec, MAX = 5000 msec
```

To avoid SPF recalucation in case of link flapping, when a new SPF calculation is required, it is delayed the DELAY interval. If a new calculation is required in the HOLD-TIME, it is delayed and the value of the HOLD-TIME increases with another HOLD-TIME. The total HOLD-TIME value cannot be larger then MAX-HOLD-TIME

```
R(config-router)# timers throttle spf DELAY HOLD-TIME MAX-HOLD-TIME
! Disabled by default.
```

### Other timers

OSPF will accept the same LSA from a neighbor only if the specified time has passed since the last time it received the LSA:

```
R(config-router)# timers lsa arrival MSEC
! Default: 1000 msec
```

If an adjacency must be reset, the router will wait for the maximum value between the RESYNC-TIMEOUT and the interface DEAD interval:

```
R(config-if)# ip ospf resync-timeout SEC
!default: 40 sec
```

If a LSA is not acknowledged, it can be retransmitted after RETRANSMIT-INTERVAL timesout:

```
R(config-if)# ip ospf retransmit-interval SEC
! Default: 5 sec
```

The serialization delay needed to send a LSA out. This value will be added to the LSA age when sending out. It should be greater on low-speed interfaces.

```
R(config-if)# ip ospf transmit-delay SEC
! Default: 1 sec
```

## Packets

OSPF uses IP protocol 89 to send OSPF packets. It sends packet as unicast or multicast to 224.0.0.5 (All OSPF Routers) and 224.0.0.6(All OSPF DRs).

### Packets used in database exchange

Packets sent at this stage are sent as unicast to each router.

* **Database Description** carries a summary description of each LSA. They are used by the receiving router to check if it has the latest LSAs. If a neighbor sees that it doesn’t have a LSA or the latest version of it, it will place the LSA on the Link State Request List
* **Link State Request** is sent to request missing LSAs
* **Link State Update** is sent in response to a LSR with the missing LSA information. As LSAs are received, they are removed from the Link State Request List. All LSAs sent in updates must be acknowledged individually, so before they are sent, they are placed in a Link State Retransmission List and are removed when they are acknowledged.
* **Acknowledgements** can be explicit – A Link State ACk is received containing the LSA Header, or implicit – an update that contains the same instance of the LSA is received

### LSA Flooding

Flooding occurs whenever a change in LSDB happens. This includes a change in an existing LSA or a new LSA. A LSU can contain multiple LSAs. When a LSA is sent to a neighbor a copy of the LSA is added to the Link State Retransmission list. The LSA is retransmitted every Rxmt interval until an ACK is received and it is removed from the Link State Retransmission list. LSUs containing retransmissions are always unicast

Acknoweldgements can be

* **implicit** – A neighhbor implcitly acknowledges an LSA by including a duplicate of the LSA in an LSU back to the originator
* **explicit** – A LSAck is sent back to the originator containing the acknowledged LSA headers
* **direct** – Sent when a duplicate LSA is received from a neighbor (it didn’t receive a previous ACK) or when the LSA’s age is MaxAge and there is no instance of the LSA in the router’s LSDB. They are always sent as unicast
* **delayed** – More LSAs can be acknowledged by a single LSAck. LSAs form multiple neighbors can be acknowledged in a single multicast LSAck. The delayed period must be less than RxmtInterval to avoid retransmission

## Path Selection

The first criteria when choosing OSPF routes is the route type, and then, for the same route type, the metric is considered:

### Route Type

1. **Intra Area (O)** – routes in the same area
2. **Inter Area (O IA)** – routes in another area, but in the same AS
3. **External Type 1 (O E1)** – External paths where the cost is calculated as cost to the ASBR + the cost assigned by the ASBR to the external route
4. **External Type 2 (O E2)** – Ignores the cost to the ASBR when calculating the metric of an external route. It is the default type for external routes!
5. **NSSA Type 1 (O N1)** – external routes from an ASBR in the same NSSA area. Both the cost to the ASBR and the cost assigned by the ASBR are taken into account
6. **NSSA Type 2 (O N2)** – external routes from an ASBR in the same NSSA area, but ignores the cost to the ASBR and considers only the cost assigned by the ASBR to the route

### OSPF Metric

The metric used is the cost of each link in the path.\


$$
OSPF_{metric} = \sum_{path}Cost
$$

$$
Cost = \frac {BW_{reference}}{BW_{interface}}
$$

$$
Cost_{default} = \frac{10^5}{BW_{interface}[kbps]} = \frac{10^8}{BW_{interface}[Kbps]}
$$

\
The reference bandwidth is 100Mbps by default, but can be changed using:

```
R(config-router)# auto-cost reference-bandwidth REFERENCE-BW
! REFERENCE-BW must be entered in Mbps. Default: 100
```

The cost of a link cannot be less than 1 and will be narrowed down to the closes integer.\
The cost can be also manually assigned per interface:

```
R(config-if)# ip ospf cost COST
```

or per neighbor:

```
R(config-if)# ip ospf neighbor cost COST
```

Of course, modifying the bandwidth of an interface will affect the cost:

```
R(config-if)# bandwidth KBPS
```

Using default values, the following costs will be obtained:

| Interface Type     | Default Bandwidth | Default OSPF Cost |
| ------------------ | ----------------- | ----------------- |
| Loopback           | 8.000.000 Kbit    | 1                 |
| Gigabit and higher | 1.000.000 Kbit    | 1                 |
| Fast Etheret       | 100.000 Kbit      | 1                 |
| Ethernet           | 10.000 Kbit       | 10                |
| Serial             | 1544 Kbit         | 64                |

## OSPF Administrative Distance

In OSFP, the default AD for each type of route is 110.\
You can modify the AD filtering on the route type:

```
R(config-router)# distance ospf {[intra-area AD-INTRA] [inter-area AD-INTER] [external AD-EX]}
```

or on the source router (the router that originated the route):

```
R(config-router)# distance AD SOURCE-ROUTER [WILDCARD [ACL]]
```

## Load Balancing

If multiple equal cost paths exist in the final set, OSPF will use them (max 16).

## Filtering

### At the ABR

To filter inter-area routes in or out of an area, use:

```
R(config-router)# area AREA filter-list prefix PREFIX-LIST {in|out}
```

### Filter via summarization

You can also filter Type 3 LSA with area range. When using the not-advertise keyword, the summarry route and its children will not be advertised as a Type 3 LSA => Same effect as “area filter-list out”

```
R(config-router)# area AREA range SUMMARY-IP MASK [not-advertise]
```

### Filter via distribute-lists

Distribute lists can only be used to filter inbound routes and only affects the local routing table, not the LSDB. Due to the Link State nature of OSFP, you cannot filter outbound routes with distribute lists.

```
! Using ACLs
R(config-router)# distribute-list ACL in [INTERFACE]
! Using Prefix-list
R(config-rotuer)# distribute-list prefix PREFIX-LIST in [INTERFCE]
! Using gateway - filter based on the source of the update:
R(config-router)# distribute-list gateway PREFIX-LIST-1 [prefix PREFIX-LIST-2] in [INTERFACE]
! Using route maps:
R(config-router)# distribute-list route-map ROUTE-MAP in [INTERFACE]
```

### Filter using stub area

See [OSPF Areas](https://nyquist.eu/ospf-areas/)

### Filter all LSAs

Used to filter routes only one way, similar to RIP passive-interface.\
On broadcast, non-broadcast and point-to-point networks you can block flooding over specified OSPF interfaces

```
R(config-if)# ip ospf database-filter all out
```

On point-to multipoint networks you can block flooding to a specified neighbor

```
R(config-if)# neighbor NEIGH-ADDR database-filter all out
```

## Summarization

You cannot summarize inside an area because every router must know all paths in an area it belongs to. Summarization can therefore be done only between areas or when redistributing. When advertising summary routes, an OSPF router will auto-generate a discard route (Summary to NULL0). This behavior can be disabled using:

```
R(config-router)# no discard route [internal | external]
! internal - will not generate NULL route for internal summaries
! external - will not generate NULL route for external summaries
```

### Summarization at the ABR

One point in the network where summarization can be configured is at an ABR. The ABR can send only one Type 3 LSA for multiple destinations in the area:

```
R(config-router)# area AREA range SUMMARY-IP MASK [cost COST] [advertise|not-advertise]
!advertise – advertises the summary route using Type 3 LSA
!not-advertise - suppresses the summary route AND the children
```

### Summarization at the ASBR

Another point in the network where summarization can be configured is at an ASBR. The ASBR can send only one Type 7 LSA for multiple destinations that are redistributed into OSPF, using:

```
R(config-router)# summary-address SUMMARY-IP MASK [not-advertise] [nssa-only] [tag TAG]
! not-advertise - suppresses the Type 7 LSA
! nssa-only - the LSAs are not sent outside the NSSA area
! tag - adds a TAG to this route
```

## Default Routes

### Default routes in normal areas

OSPF routers do not advertise default routes by default, but you can change this with:

```
R(config-router)# default-information originate [always] [metric METRIC] [metric-type {1|2}] [route-map ROUTE-MAP]
```

The router will inject a Type 5 LSA advertising a default route if it already has a default route in the routing table.\
Use **always** to generate a Type 5 LSA advertising a default route even when there is no default route in the routing table. If the keyword is not used, the router will never generate a default route unless it has one in the routing table.

Conditional advertising of the route can be achieved using a route-map.

Redistribute static subnets command is used to redistribute static routes in OSPF. However static default route is not injected in to OSPF topology database this way

### Default routes in Stub areas

ABRs advertise default routes into a stub area instead of the Type 4 and 5 LSAs

```
R(config-router)# area AREA default-cost COST
!Assigns a specific cost to the default summary route used for stub areas
```

### Default routes in Totally Stubby Areas

ABRs inject only a Type 3 LSA advertising a default route into a Totally Stubby Area instead of the Type 3, 4 and 5 LSAs

### Default routes in Not So Stubby Areas

ABRs do not advertise default routes into NSSAs by default. To inject a Type 7 LSA advertising a default route into the NSSA, use:

```
R(config-router)# area AREA nssa default-information-orginate
```

### Default routes in NSSA Totally Stubby Areas

ABRs advertise default routes into NSSA Totally Stubby Areas by default as Type 3 LSA
