# More BGP

## Route Dampening

It is used to stop unstable routes from being forwarded throughout the network. When a route flaps, a penalty is assigned to the route (Default: 1000 per flap). A timer called Half-Life is used to reduce the penalty value to half (Default: 15 min). If the penalty value exceeds the suppress limit, the route is no longer advertised (Default: 2000). The route continues to be suppressed until the Half-Life reduces the penalty below the reuse limit (Default: 750). A route cannot be suppressed more than the Maximum Suppress Time (60 min or 5 \* Half-Life).\
Dampening can be enabled globally:

```
R(config-router)# bgp dampening [PARAMS]
```

or only for some routes matched in a route-map:\
R(config-router)# set dampening PARAMS\
! Must set PARAMS. It won’t use default settings\
When setting dampening with a route-map, define dampening parameters in the route-map.\
You can verify dampening with:

```
R# show ip bgp dampening parametrs
```

Dampened prefixes can be manually cleared, using:

```
R# clear ip bgp dampening PREFIX NETMASK
```

However, this will not clear the penalty value so if a new flap occurs, the route will probably be immediately dampened. To clear the penalty value, use:

```
R# clear ip bgp NEIGH-ADDR flap-statistics
```

## Backdoor networks

By default, an external BGP learned route is preferred due to the lower AD (20) to an IGP learned route. If you would like to prefer the route learned via an IGP, use the Backdoor command on the network:

```
R(config-router)# network NETWORK-ADDR backdoor
```

This command will modify the AD of the BGP learned route to 200, making it less preferred over other IGP routes.

## Fast Fallover

### Fast External Fallover

For eBGP peers that are directly connected, the router will bring down the neighbor relationship if the interface status goes down. In case of flapping interfaces, you may want to keep the neighbor relationship up until the dead timer expires.

```
R(config-router)# [no] bgp fast-external-fallover
! Default: on
```

The command can be issued per interface, with:

```
R(config-router)# ip bgp fast-external-fallover {permit|deny}
! permit - enables fast-external-fallover
! deny - disables fast-external-fallover
```

### Internal Fallover

For iBGP peers you can also enable fast fall-over with the command:

```
R(config-router)# neighbor NEIGH-ADDR fall-over [route-map ROUTE-MAP]
! ROUTE-MAP can be used for conditional deactivation of a session
```

However, this is usually not a desired behavior.

## ORF – Outbound Route Filtering

Sending a lot of updates which are filtered inbound by a neighbor is unnecessary but there was no way for a router to know how its neighbors would handle the routes. With the introduction of the ORF feature, 2 ORF-capable router can exchange information regarding their inbound filters, so that the sending router can filter them in the outbound direction.\
To enable this feature, 2 neighbor routers must be configured with:

```
R(config)# neighbor NEIGH-ADDR capability orf prefix-list {both|send|receive}
! both     Capability to SEND and RECEIVE the ORF to/from this neighbor
! receive  Capability to RECEIVE the ORF from this neighbor
! send     Capability to SEND the ORF to this neighbor
```

After the command is enabled on both routers, they will exchange information and will filter outbound updates with the same prefix list as the filter set on the other peer in the inbound direction, therefor optimizing the bandwidth usage.

## Local AS

For temporary situation when a company migrates from one AS number to another, it is necessary to be able to change the AS number used on a per-neighbor basis.\
This can be done using the command:

```
R(config-router)# neighbor NEIGH-ADDR local-as NEW-AS [no-prepend [replace-as [dual-as]]]
```

Now, the router will make connections to this neighbor as if it is running BGP in the NEW-AS.

When the command is entered only with the local-as keyword, then the router will do the following:

* The AS\_PATH of the routes received from this neighbor will be prepended with the NEW\_AS. When they are sent out to other eBGP peers, they will also be prepended with the OLD\_AS
* The AS\_PATH of the routes sent to this neighbor will be prepended with both the NEW\_AS and the OLD\_AS, with the NEW\_AS appearing first in the AS\_PATH

When the command is entered with the “no-prepend” keyword, the router will do the following:

* The AS\_PATH of the routes received from this neighbor will not be prepended with the NEW\_AS. When they are sent out to other eBGP peers, they will only be prepended with the OLD\_AS
* The AS\_PATH of the routes sent to this neighbor will be prepended with both the NEW\_AS and the OLD\_AS, with the NEW\_AS appearing first in the AS\_PATH

When the command is entered with the “no-prepend replace-as” keywords, the router will do the following:

* The AS\_PATH of the routes received from this neighbor will not be prepended with the NEW\_AS. When they are sent out to other eBGP peers, they will only be prepended with the OLD\_AS
* The AS\_PATH of the routes sent to this neighbor will be prepended only with the NEW\_AS, skipping the OLD\_AS

When using also the “dual-as” keyword, the router will accept peering with this neighbor on both AS-NUMBERS, making it easy to migrate from one AS-NUMBER to another.

## Maximum prefixes

The internet BGP tabel size is huge and if you receive such a large number of routes from a neighbor, you’re router might not have the amount of memory to manage it. This is why you can configure a maximum number of prefixes that you are willing to receive from your neighbor. When the Maximum number of prefixes is reached, the session is shut down.

```
R(config-router)# neighbor NEIGH-ADDR maximum-prefix MAX [TH] [restart TIMER] [warning-only]
! MAX = maximum number of routes
! TH = When the number of received routes reaches TH% of MAX, the router generates a warning. Default: 75
! restart TIMER = The session will be reestablished after the TIMER expires. If not configured, it will remain shutdown.
! warning-only = doesn't shutdown, but issues a warnign (syslog)
```
