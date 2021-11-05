# Route Redistribution

## Route Redistribution

You can redistribute routes from one routing process to another using the redistribute command inside the destination routing process:

```
R(config-router)#redistribute SRC [PROC|AS] [metric METRIC] [route-map ROUTE-MAP] [OSPF-OPTIONS]
! SRC: rip, eigrp, ospf, isis, bgp, static, connected, odr
! PROC|AS: used for OSPF and EIGRP
! METRIC: seed metric into the DESTINATION process
! ROUTE-MAP: used for conditional redistribution
! OSPF-OPTIONS: options specific to OSPF
```

When a routing protocol starts, it automatically redistributes connected routes that are matched by the network command. This also happens for static routes that point to an interface if they are matched by a network command. These routes are considered to be directly connected and will be redistributed without the **static** keyword.

### Redistributing into OSPF

When redistributing into OSPF, automatic classless summarization takes place, unless the **subnets** keyword is specified:

```
R(config)# router ospf PROC
R(config-router)# redistribute [SRC [PROC|AS]] subnets
```

### Redistributing from OSPF

When redistributing OSPF routes, you can select the types of routes that are redistributed.

```
R(config-router)# redistribute ospf PROC match {internal|external {1|2}|nssa-external {1|2}}
! using just external means both 1 and 2
```

When redistributing OSPF routes into BGP, the default behavior is to redistribute only OSPF intra-area and inter-area routes. You must use the **external** keyword to enable the redistribution of OSPF external routes.

### Redistributing BGP into IGP

When redistributing BGP, only eBGP routes are redistributed by default. You can modify this behavior if you use the command:

```
R(config-router)# bgp redistribute-internal
```

## Seed metric

By default only when redistributing from one EIGRP process to another or from one OSPF process to another, the metric values can be used from the first to the second.

### RIP

When redistributing from any other protocol to RIP, the **metric transparent** keyword can make RIP use the numerical value of the metric from the other protocol. If this value is greater than 15 (usually), the route will be considered unreachable in RIP.

```
R(config)# router rip
R(config-router)# redistribute [SRC [PROC|AS]] metric {transparent|VALUE}
```

The router that performs redistribution into RIPt will send updates to other RIP routers using the seed metric. When those routers will send updates to the next hop, they will add one more hop to the metric.

### OSPF

When redistributing from any other protocol to OSPF, the default metric value is 20 with a metric type of E2.

```
R(config)# router ospf
R(config-router)# redistribute [SRC [PROC|AS]] metric VALUE [metric-type {type1|type2}]
```

### EIGRP

When redistributing from any other protocol to EIGRP, the metric cannot be auto-converted. You must use the EIGRP **metric** keyword with the redistribute command.

```
R(config)# router eigrp
R(config-router)# redistribute [SRC [PROC|AS]] metric BW DEL REL LOAD MTU
```

### Default metric

You can alse set a default metric regardless of the route source:

```
R(config-router)# default-metric {VALUE|BW DEL REL LOAD MTU}
! BW DEL REL LOAD MTU: used for EIGRP
! VALUE: used for other protocols
```

The metric used with the redistribute command overrides this default value.

## Conditional Redistribution

Conditional redistribution can be achieved by using a route-map. Only routes that are allowed in the route-map are redistributed into the protocol. Also, the set commands in the route-map are applied to the redistributed routes.\
A route map is defined as a series of entries identified by a sequence number (SEQ). Routes that are matched by a permitting entry are redistributed, while routes that are matched by a denying entry are not redistributed.

### Match rules

Each route is passed through each entry until a match is found, if no match is found, the default is to not redistribute the route. You can change this by using an entry with no match commands that would match any route.

```
R(config)# route-map REDIST-MAP {permit|deny} [SEQ]
! default: permit 10
```

In each entry a set of match rules define the routes that are matched by the entry. If multiple match rules are defined, a packet must pass all of them (logical AND). Some match rules can have multiple entries (for example multiple ACLs in one line). In this case, a route is considered to be matched if it is matched by at least one entry in the match rule (logical OR). Also, by entering the same match type with different conditions (e.g. multiple **match ip address ACL**) they will be converted to one match entry with multiple entries – therefore the logical OR will apply.

#### **General match rule**

```
R(config-route-map)# match interface INTERFACE
! Matches routes that have the next hop on the defined INTERFACE
R(config-route-map)# match {ip|ipv6} address ACL
! Matches routes that point to a destination matched by the ACL
R(config-route-map)# match {ip|ipv6} prefix-list PREFIX-LIST
! Matches routes that point to a destination matched by the PREFIX-LIST
! A PREFIX-LIST can also match the network mask
R(config-route-map)# match {ip|ipv6} next-hop ACL
! Matches routes that have a next-hop matched by the ACL
R(config-route-map)# match {ip|ipv6} route-source {ACL|prefix-list PREFIX}
! Matches routes advertised by an IP matched in the ACL/prefix-list
R(config-route-map)# match tag TAG
! Matched routes tagged with TAG
R(config-route-map)# match metric VALUE [+- D]
! Matches routes with a specific VALUE or in the range VALUE-D VALUE+D
```

#### **IGP specific**

We saw before that we can use an extended ACL to match a route using:

```
R(config-route-map)# match {ip|ipv6} address ACL
! Matches routes that point to a destination matched by the ACL
```

When dealing with IGP routes, the extended ACL will be interpreted as follows: The SOURCE in the extended ACL will match the update source, while the DESTINATION will match the route destination.\
E.g:

```
R(config)# access-list 101 permit host SRC host DST
! will match routes advertised by SRC for the destination DST
```

Of course, any binary math can still be used.\
Other rules:

```
R(config-route-map)# match route-type {external|internal|nssa-external}
! OSPF only
R(config-route-map)# match route-type external
! EIGRP only
```

#### **BGP specific**

We saw before that we can use an extended ACL to match a route using:

```
R(config-route-map)# match {ip|ipv6} address ACL
! Matches routes that point to a destination matched by the ACL
```

When dealing with BGP routes, the extended ACL will be interpreted as follows: The SRC in the extended ACL will match the destination address, while the DST will match the network mask.\
E.g:

```
R(config)# access-list 101 permit host 10.0.0.0 host 255.255.255.0
! will match the route 10.0.0.0/24
```

Of course, any binary math can still be used.

```
R(config-route-map)# match route-type local
! Locally originated BGP routes
R(config-route-map)# match as-path PATH-LIST
R(config-route-map)# match community COMM [exact-match]
R(config-route-map)# match local-preference LP
```

### Set rules

Set commands:

```
! Set metric
R(config-route-map)# set metric {VALUE|BW DEL REL LOAD MTU}
! Set OSPF metric-type
R(config-route-map)# set metric-type {type-1|type-2}
! Set automatic tags
R(config-route-map)# set automatic-tag
! Set manual tag
R(config-route-map)# set tag TAG
! BGP Specific:
R(config-route-map)# set community COMMUNITY
R(config-route-map)# set local-preference LP
R(config-route-map)# set weight WEIGHT
R(config-route-map)# set origin {igp|egp AS|incomplete}
R(config-route-map)# set as-path [prepend] AS-LIST
```
