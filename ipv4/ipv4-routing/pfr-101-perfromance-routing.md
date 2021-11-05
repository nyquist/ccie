# PfR 101 – Perfromance Routing

## PfR Technology

PfR stands for Performance Routing, but the feature was first called OER (Optimized Edge Routing). This is why most commands still start with the **oer** keyword. The idea behind PfR is to have a controlling entity (Master Controller) that takes over routing decisions for one or more Border Routers, in order to deliver traffic according to a specified policy. The MC will monitor network statistics (NetFlow, IP SLA) and will chose the best exit point for each type of traffic. PfR is used to enhance the operation of traditional routing protocols by using real time network performance information.

## PfR Components

Each PfR implementation requires the following minimum components:

* 1x Border Router
* 1x Master Controller
* 1x Internal Interface
* 2x External Interfaces

The MC and the BR can be configured on the same device. They both need to have a matching key-chain defined, that will secure communication between them, even if running on the same device:

```
R(config)# key chain PFR-KEY
R(config-key)# key KEY-ID
R(config-key-string)# key-string PASSWORD
```

### Border Router

To define a router as OER BR, use:

```
R(config)# oer border
```

Inside the **oer-br** configuration mode, you can define the interface used for connection to the MC and the IP address of the MC:

```
R(config-oer-br)# local INTERFACE 
! usually Loopback
R(config-oer-br)# master MASTER-IP key-chain PFR-KEY
```

Normally, OER uses TCP port 3949 on both the MC and BR. This can be changed with the following command, but it has to match on both MC and BR:

```
R(config-oer-br)# port PORT-NUMBER
```

To enable OER to generate syslog messages, use:

```
R(config-oer-br)# logging
```

To stop the OER process, without deleting the configuration, use:

```
R(config-oer-br)# shutdown
```

### Master Controller

To define a router as the MC, just use:

```
R(config)# oer master
```

Here are a few general commands used on the MC:

```
R(config-oer-mc)# port PORT-NUMBER
! See above. It has to match the same command on the BR
```

The MC and BRs exchange keepalives at a rate that is configurable (5 sec by default). If 3 keepalives are missed, the MC considers the BR down

```
R(config-oer-mc)#keepalive SEC
! Defualt: 5
```

To enable OER to generate syslog messages, use:

```
R(config-oer-mc)# logging
```

To stop the OER process, without deleting the configuration, use:

```
R(config-oer-mc)# shutdown
```

The total number of prefixes that OER can monitor can be customized with the command:

```
R(config-oer-mc)#max prefix total TOTAL learn LEARNED
! Default: TOTAL=5000, LEARNED=2500
```

To verify the status of the MC, use:

```
R# show oer master
! Major Version must match between MC and BR
! Master controller must have a higher or equal minor version
```

#### **Defining BRs**

In the **oer-mc** config mode you can then define the BRs that the MC will control:

```
R(config-oer-mc)# border BORDER-IP key-chain PFR-KEY
! The session is always initiated by the Border Router, so it doesn't need a local interface to be defined
```

Once you define the BR on the MC, you can then select which interfaces on the BR will be part of the PfR setup. Over all BRs you will need to have at least 1 internal and 2 external interfaces.

#### **Internal Interfaces**

An internal interface is used for passive monitoring (using netflow) and normally should be placed on the internal network between the MC and the BRs. To define an internal interface, use:

```
R(config-oer-mc-br)# interface IN-INTERFACE internal
! Config done on the MC
```

#### **External Interfaces**

An external interface is used to forward the traffic and the performance is actively monitored (using IP SLA) by the MC. To define an external interface, use:

```
R(config-oer-mc-br)# interface EX-INTERFACE external
```

For each external interface you can add additional configuration:

```
R(config-oer-mc-br-if)# cost-minimzation ...
! Used to configure the actual cost of routing on one exit interface (through one ISP).
R(config-oer-mc-br-if)# link-group GROUP1 [GROUP2 [GROUP3]]
! Used to add the interface to a link group, which can later be used as a preferred exit point
! An interface can be part of up to 3 Link Groups.
R(config-oer-mc-br-if)# max-xmit-utilization {absolut KBPS|percentage PERCENT}
! The MC will forward traffic on this link as long as the utilization is below the defined value.
! Default: 75%
R(config-oer-mc-br-if)# maximum utilization receive {absolut KBPS|percentage PERCENT}
! Defines the maximum allowed received traffic. Default 75%
R(config-oer-mc-br-if)# downgrade bgp community COMMUNITY
! Adds a BGP community to updates advertising this link in order to downgrade it's usage.
```

Under the global MC configuration mode, you can also define a parameter that is used for proper load-balancing.

```
R(config-oer-mc)# max-range-utilization percent PERCENT
! Default: 20
```

The MC will look at the utilization value for all exit interfaces and if the difference between highest and lowest is more than the above configured parameter, than it considers that the links are not properly balanced so it will try to move some of the traffic from one exit to another.\
There is a similar parmeter for incomint utilization:

```
R(config-oer-mc)# max range receive percent PERCENT
! Default: 20
```

## PfR Policies

PfR uses for its operation a set of rules, called a policy. There is a default policy that is configurable from withing the oer master configuration mode, but you can create your own policies using oer-maps, which are somewhat similar to route-maps or policy-maps.

```
R(config)# oer-map PFR-POLICY [seq SEQ]
```

Inside each entry (each SEQ) you can have **only one match command** and several set commands that override the defaults. The match commands are used to filter the type of traffic that this policy applies to. The filtering can be statically configured or learned dynamically. See [Learning](broken-reference).

```
R(config-oer-map)# match  ?
  ip             IP specific information
  oer learn      Match dynamically learned OER prefixes
  traffic-class  Specify Traffic class
```

After matching the traffic, you define the actual policy in a series of set commands that override the values of the default policy:

```
R(config-oer-map)# set ...
```

In the end, apply the policy on the MC:

```
R(config-oer-mc)#policy-rules PFR-POLICY
```

You can only apply a single policy-map. If you run the command again with a new policy, it will use the new policy instead of the old policy.\
You can see the configured OER polices using:

```
R# show oer master policy [default|PFR-POLICY]
```

## PfR Stages

PfR uses 5 stages:

1. **Learning (aka Profiling)**: The MC tells the BR to learn the Traffic Classes. These classes can be dynamically learned (via NetFlow) or statically learned (manual configuration).
2. **Measuring Performance**: The BRs collect Traffic Class statistics
3. **Apply Policies**: Use measurements to determine whether a Traffic Class is out of policy (OOP) and if an alternate path can be found.
4. **Enforcing (traffic re-routing)**: Enforcing is done by injecting BGP or static routes, or adding Policy Based Routing (PBR) configuration.
5. **Verification**: Verify that the new route match the policy

### Learning

#### **Static Learning**

For static configuration use one of the following commands:

```
R(config-oer-map)# match ip address {access-list ACL | prefix-list PREFIX}
R(config-oer-map)# match traffic-class application APPLICATION
R(config-oer-map)# match traffic-class {access-list ACL | prefix-list PREFIX}
```

#### **Dynamic Learning**

To configure the policy to use dynamically learned traffic classes, use:

```
R(config-oer-map)# match oer learn {delay|throughput|inside|list LEARN-LIST}
```

Now, you will have to define how learning actually works inside the oer master configuration mode.\
First, to enable dynamic learning, use the following command on the oer master:

```
R(config-oer-mc)# learn
```

Now you can enable learning based on Top Talkers (throughput) and Top Delay

```
R(config-oer-mc-learn)# delay
! Enables learning by Top Delay
R(config-oer-mc-learn)# throughput
! Enable learning by Top Talkers (throughput)
R(config-oer-mc-learn)# inside bgp
! Learns inside prefixes advertised through BGP 
```

Additional learning settings can be configured:

```
R(config-oer-mc-learn)# ?
  aggregation-type   Type of prefix to aggregate. Default: prefix-len 24
  expire             Set expiry criteria for learned prefixes. Default: 720 min
  monitor-period     Period to monitor prefix for learning. Default: 5 min
  periodic-interval  Interval before learning restarts. Default: 120 min
  prefixes           Number of prefixes to learn. Default: 100
```

By default, the MC monitors for 5 minutes and then pauses for 120 min, meaning no routes change for another 120 min. To see results quicker, lower (drastically) these values.\
Traffic lists can be further defined to only learn and categorize specific types of traffic:

```
R(config-oer-mc-learn)# list seq SEQ refname LEARN-LIST
Next, define the traffic classes (more exactly what exactly makes this Traffic Class):
R(config-oer-mc-learn-list)#traffic-class {access-list ACL ...|application ...|prefix-list PREFIX-LIST}
For each least you can specify the method of learning:
R(config-oer-mc-learn-list)#{delay|throughput|rsvp}
You can see the values currently used for Learning with:
R# show oer master | se Learn
				
```

### Measuring

Monitoring the policy can be done passively, actively or using both passive and active monitoring. Passive mode uses Netflow informatation from the BRs (No actual netflow config is needed). Active monitoring uses IP SLA probes that are configured on the MC, but are sourced from the BRs. The other options (fast, both, active throughput) use both active and passive monitoring. To configure the monitoring mode, use:

```
! In the Default Policy:
R(config-oer-mc)# mode monitor {active [throughput] | passive | fast | both}
! In a custome OER-MAP:
R(config-oer-map)# set mode monitor {active [throughput] | passive | fast | both}
To define SLA probes:
! In the Default Policy:
R(config-oer-mc)# active-probe {echo|jitter|tcp-conn|udp-echp} OPTIONS
R(config-oer-mc)# probe frequency SEC
! Default: 60
! In a custom OER-MAP:
R(config-oer-map)# set active-probe {echo|jitter|tcp-conn|udp-echp} OPTIONS
R(config-oer-map)# set probe frequency SEC
! Default: 60
To define the frequency of traceroute probes, use:
! In the Default Policy:
R(config-oer-mc)# traceroute probe delay MSEC
! Default: 1000 msec = 1 sec
! In a custom OER-MAP:
R(config-oer-map)# set traceroute probe delay MSEC
! Default: 1000 msec = 1 sec
```

### Applying Policy

In this stage, OER compares the measurements with thresholds defined in the policy in use. If the perfromance doesn't fit the requirements, OER considers the traffic to be OOPOLICY (Out of Policy) and makes a decision to move it on another exit. To configure the thresholds, use:

```
! In the Default Policy:
R(config-oer-mc)# {delay|loss|jitter|mos|unreachable} [threshold MIN| relative PERCENTAGE] ...
! In a custom OER-MAP:
R(config-oer-map)# set {delay|loss|jitter|mos|unreachable} [threshold MIN| relative PERCENTAGE] ...
The router will try to find a new exit interface for a Traffic Class if it goes OOPOLICY, but also after a fixed period, regardless of the status. This timer is disabled by default but it can be set with:
! In the Default Policy:
R(config-oer-mc)# period SEC
! In a custom OER-MAP:
R(config-oer-map)# set period SEC
When a Traffic Class is moved to a new exit, it will not be changed again until a holddown timer expires. This timer is configured with:
! In the Default Policy:
R(config-oer-mc)# holddown SEC
! Default: 300
! In a custom OER-MAP:
R(config-oer-map)# set holddown SEC
! Default: 300 
Once a Traffic Class becomes OOPOLICY it has to wait a backoff time before beeing moved to an INPOLICY state. This backoff time starts at the MIN value and increases with STEP value for each time the MC tries to move it INPOLICY, but fails. It can't grow more than MAX.
! In the Default Policy:
R(config-oer-mc)# backoff MIN MAX STEP
! Default: MIN=300, MAX=3000, STEP=300
! In a custom OER-MAP:
R(config-oer-map)# backoff MIN MAX STEP
! Default: MIN=300, MAX=3000, STEP=300
When chosing an INPOLICY exit there may be multiple available choices. The MC will chose it based on the order of the resolvers (based on priority. Lowest goes first). Priority 0 is always reachability, so traffic will not be blackholed. To define the order of the resolvers, use:
! In the Default Policy:
R(config-oer-mc)# resolve RESOLVER priority PRI [variance VAR]
! Available Resolvers:
!  cost                         Specify PfR cost policy resolver settings
!  delay                        Specify PfR delay policy resolver setting
!  jitter                       Specify PfR jitter policy resolver settings
!  loss                         Specify PfR loss policy resolver settings
!  mos                          Specify PfR MOS policy resolver settings
!  range                        Specify PfR range policy resolver settings
!  utilization                  Specify PfR utilization policy resolver settings
R(config-oer-map)# set resolve RESOLVER priority PRI [variance VAR]
By default, after Reachability (priority 0), the MC uses Delay (priority 11) and utilization (priority 12) to solve the problem.  The Variance value is used to consider equal multiple values within the same variance. (E.g. Delay values of 300 and 320 are considered equal if the variance is at least 20).
Then, if there are multiple exits that would move the Traffic Class INPOLICY, the router can select the best exit, or any of the good enough exits, based on this command:
! In the Default Policy:
R(config-oer-mc)# mode select-exit {good|best}
! In a custom OER-MAP:
R(config-oer-map)# select-exit {good|best}
! Default: good
When selecting the exit, routers can use by default any external interface. But for some types of traffic, you can select the exit interface on a subgroup of the external interface. Remember that each external interface can be part of up to 3 LINK-GROUPS. If GROUP1 is not available, then GROUP2 can be used.
R(config-oer-map)# set link-group GROUP1 [fallback GROUP2]
For some types of traffic you can select a more static exit strategy:
R(config-oer-map)# set next-hop NEXT-HOP-IP
To drop traffic, just use:
R(config-oer-map)# set interface Null0
```

### Enforcing

The default mode of operation of OER/PfR is "observe mode". In this mode, the MC does what it would normally do, except sending commands to the BRs. This mode is ususually used to see how OER would perform without doing any actual modifications to the traffic paths. To change the mode of operation, use:

```
! In the Default Policy:
R(config-oer-mc)# mode route [control|observe]
! In a custom OER-MAP:
R(config-oer-map)# set mode route [control|observe]
! Default: Observe
```

For traffic classes that are defined using a prefix only, the prefix reachability information used in traditional routing can be manipulated. Protocols such as Border Gateway Protocol (BGP) or RIP are used to announce or remove the prefix reachability information by introducing or deleting a route and its appropriate cost metrics. For traffic classes that are defined by an application in which a prefix and additional packets matching criteria are specified, OER cannot employ traditional routing protocols because routing protocols communicate the reachability of the prefix only and the control becomes device specific and not network specific. This device specific control is implemented by OER using policy-based routing (PBR) functionality. If the traffic in this scenario has to be routed out to a different device, the remote border router should be a single hop away or a tunnel interface that makes the remote border router look like a single hop. PfR uses the concept of parent Route. It won't route traffic on a link unless you have a route for it using that link. In older version of IOS (before 12.4(24)), PFR supported as parent routes only BGP and static routes (even those static floating routes that don't make it into the routing table). With newer versions that support PIRO (Performance Routing Protocol Independent Route Optimization), it should be able to use any type of route

### Verification

PfR uses NetFlow to automatically verify the route control. The MC expects a Netflow update for the traffic class from the new link interface and ignores Netflow updates from the previous path. If a Netflow update does not appear after two minutes, the master controller moves the traffic class into the default state. A traffic class is placed in the default state when it is not under PfR control.
