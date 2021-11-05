# FHRP 101

## HSRP

HSRP provides a virtual MAC address and a virtual IP address that is shared among a group of routers in order to have a HA infrastructure for the default gateway in a subnet.

### Starting HSRP

```
R(config-if)# standby [GROUP] [ip GROUP-IP [secondary]]
! Default GROUP = 0
! The Group IP can be discovered from other HSRP routers
! the GROUP-IP can't be the interface IP
```

### Timers

```
R(config-if)# standby delay minimum SEC reload SEC
! mimium SEC = time to wait after interface comes up, before HSRP is started
! reload SEC = time to wait after router reloads, before HSRP is started
R(config-if)# standby [GROUP] [msec] HELLO-TIME HOLD-TIME 
! default HELLO-TIME = 3 sec
! default HOLD-TiME = 10 sec
```

Timers are usually learned from the active router. Millisecond timers can only be learned when using version 2. Otherwise, they must be configured on all routers.

### Election process

An election process takes place, where the primary router is elected. Only one device will be elected as the primary router. It will receive and forward packets destined for the Group IP. At the same time, a standby router is elected. It will monitor if the primary router is still reachable and if the active router fails, the standby router takes over and a new router is elected to be the standby router.\
The router with the highest priority will be elected as the primary router. The default priority or all routers is 100. In case of a tie, the router with the highest IP Address will becom the primary router.

```
R(config-if)# standby GROUP priority PRI
! Default: 100
```

By default if a router with higher priority comes up, it will not become the primary router unless it is configured for preemption:

```
R(config-if)# standby GROUP preempt [delay {minimum | reload | sync} SEC]
! timers are used to delay the preemption process
```

A standby router with equal priority but a higher IP address will still not preempt the primary router.

### Tracking

HSRP can track an interface or an object and will reduce the priority with a configurable value when the interface or the object state goes down:

```
R(config-if)# standby GROUP track INTERFACE [DECREMENT]
R(config-if)# standby GROUP track OBJECT [decrement DECREMENT]
! Default DECREMENT = 10
```

HSRP can also track other objects using:

### Authentication

HSRP authentication can use clear text or md5 hashes. For md5, you can use ca key-chain or a key-string:

```
R(config)# standby GROUP authentication {text STRING | key-string STRING| key-chain CHAIN}
```

### Versions

HSRPv1 sends multicast messages to 224.0.0.2, while HSRPv2 sends multicast messages to 224.0.0.102. HSRPv1 permits HSRP group numbers in the range 0-255, while HSRPv2 permits group numbers in the range 0-4095. HSRPv1 uses the virtual MAC format of 0000.0c07.acXX, where XX is the group number, while HSRPv2 uses the virtual MAC format of 0000.0C9F.FXXX – where XXX is the group number.

```
R(config)# standby version {1|2}
```

### HSRP Message

While talking to each other, HSRP enabled rotuers use the following messages:

* **Coup** When a standby router wants to assume the function of the active router, it sends a coup message.
* **Hello** The hello message conveys to other HSRP routers the HSRP priority and state information of the router.
* **Resign** A router that is the active router sends this message when it is about to shut down or when a router that has a higher priority sends a hello or coup message.

### HSRP States

```
R# show standby [brief]
```

* **Active** – The router is performing packet-transfer functions
* **Init or Disabled** – The router is not yet ready or able to participate in HSRP, possibly because the associated interface is not up. HSRP groups configured on other routers on the network that are learned via snooping are displayed as being in the Init state. Locally configured groups with an interface that is down or groups without a specified interface IP address appear in the Init state
* **Learn** – The router has not determined the virtual IP address and has not yet seen an authenticated hello message from the active router. In this state, the router still waits to hear from the active router.
* **Listen** – The router is receiving hello messages.
* **Speak** – The router is sending and receiving hello messages
* **Standby** – The router is prepared to assume packet-transfer functions if the active router fails

## VRRP

VRRP is an open standards implementation that is very similar to HSRP. It uses the terms master/backup instead of primary/standby. VRRP uses a virtual mac address in the format 0000.5E00.01XX, where XX is the group number.\
Most configurations are similar to HSRP, except they start with the vrrp keyword:

```
R(config-if)# vrrp GROUP ip GROUP-IP [secondary]
! the GROUP-IP can be the interface IP
```

VRRP advertisements are sent every second

```
R(config-if)# vrrp GROUP timers advertise [msec] INTERVAL
! The backup routers can learn the advertise interval from the Master Router:
R(config-if)# vrrp GROUP timers learn
```

VRRP can only track objects, not interfaces:

```
R(config-if)# vrrp GROUP track OBJECT [decrement DECREMENT]
```

Also, VRRP is preemptive by default, which is different than HSRP.

## GLBP

GLBP is a Cisco proprietary protocol. The advantage of GLBP is that it additionally provides load balancing over multiple routers (gateways) using a single virtual IP address and multiple virtual MAC addresses. The forwarding load is shared among all routers in a GLBP group rather than being handled by a single router while the other routers stand idle.\
An AVG (Active Virtual Gateway) will be elected using the same mechanics as the HSRP primary or the VRRP master router. The difference is that AVG’s role is to maintain a list of maximum 4 AVF (Active Virtual Forwarder) and assignes a MAC address to them, in the format 0007.b4XX.XXYY (XXXX = GLBP Group, YY – VF Number). The AVG will reply to ARP requests with the MAC address assigned to the AVFs, thus achieving load balancing.\
A router that is assigned a MAC address will be a primary AVF, while the other routers can take over the MAC address if the primary AVF fails.\
Most GLBP configurations are similar to HSRP and VRRP

```
R(config-if)# glbp GROUP ip [GROUP-IP [secondary]]
```

On the AVG you can set the load-balancing method, using:

```
R(config-if)# glbp GROUP load-balancing [host-dependent | round-robin | weighted]
! round-robin - the next available MAC is used (default)
! weighted - proportionally to the weight
! host-dependent - each client will always receive the same MAC in the ARP reply
```

When using weighted load-balancing you can define a WEIGHT for each router:

```
R(config-if)# glbp GROUP weighting WEIGHT [lower LOWER][upper UPPER]
! A router will stop forwarding when the weight drops under the LOWER threshold
! A router will start forwarding when the weight becomes higher than the UPPER threshold
R(config-if)# glbp GROUP weighting track OBJECT [decrement DECREMENT]
! A tracked object will decrement the weight 
```

Premption is off by default for AVG, but is on by default for AVF, with a delay of 30 seconds.

## GDP

Gateway Discovery Protocol is a feature that enables a host to listen for routing protocol advertisements and select a default gateway. A router must disable ip routing before using GDP:

```
R(config)# no ip routing
R(config)#ip gdp {rip|eigrp|irdp [multicast]}
! For RIP, only v1 is supported
```

To verify, use on the host:

```
R# sh ip route
```

### IRDP

Besides listening to RIP or EIGRP messages, GDP can use IRDP to discover available gateways. In order to configure a router to send IRDP (ICMP Router Discovery Protocol) messages, use:

```
R(config-if)# ip irdp
R(config-if)# ip irdp address IP-ADDR PREFERENCE
```

There is no preemption with IRDP. Instead, the PREFERENCE value is used only to chose between different addresses advertised by the oldest router (if it is configured to send advertise several addresses). A host will choose as default gateway the address with the lowest positive preference, or if they are all negative, the lowest negative.

IRDP messages are sent as broadcasts by default, but the router can be configured to send them as multicast to 224.0.0.1

```
R(config-if)# ip irdp multicast
```

The hosts will chose the IRDP\
To verify, use on the router advertising IRDP:

```
R# sh ip irdp [INTERFACE]
```
