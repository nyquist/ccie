# WCCP 101

WCCP support on Cisco router is a feature used to intercept traffic (usually HTTP requests) and redirect it to a cluster of content engines. This feature is mainly used for webcaching or WAAS implementations. The WCCP protocol is a control protocol used by router and Content Engines to communicate with each other. It uses UDP 2049.\
The main idea is that a router(v1) or more(v2) can intercept traffic (HTTP for v1, TCP/UDP for v2) and instead of sending it to the intended destination, it redirects it to a Content Engine that might have a cached version of the destination’s response. If it doesn’t, it will create a new request, get the response from the original destination, cache it and send it back to the router, destined to the client.\
The redirection can be done by encapsulating the traffic in a GRE tunnel or by L2-redirect, which is done by changing the L2 address of the frames so they reach the Content Engine instead (requires the router and the CE to be directly connected). Return traffic can use the same methods, but they don’t have to match.

## WCCP Versions

### Version 1

WCCPv1 supports only the redirection of HTTP (TCP 80) traffic. Another important limitation is that only 1 router can be used to service a group of Content Engines\
The Content Engines are configured with the IP address of the router. They can now sende WCCP messages informing it about their presence. When the router has a stable view of the CE Cluster, it sends this information back to the Content Engines. They will chose one CE (lowest IP address) to act as the Lead CE. It will tell the router how the traffic will be shared between CEs.

```
R(config)# ip wccp version 1
! Default 2
```

### Version 2

WCCPv2 supports multiple routers (up to 32) and multiple Content Engines (up to 32) used for the same Service Group. The Content Engines must signal their presence to the routers. This is done either with unicast (each CE must be configured with a list of routers) or with multicast (Each CE is configured with 1 multicast address for each Service Group. The routers must also be configured to listen for that multicast address). When the routers have a stable image of the CE Cluster, they send that information back and the Content Engines chose a CE Lead (lowest IP) just like in WCCPv1.\
Besides HTTP (TCP 80), WCCPv2 supports the redirection of other types of traffic. It also provides a mechanism for MD5 authentication between routers and CEs

```
R(config)# ip wccp version 2
! Default 2
```

## Service Groups

A Service Group represens all the Routers and Content Engines that are used to service one type of traffic.\
With WCCPv1, only the default “web-cache” Service Group can be used.\
With WCCPv2, besides “web-cache”, multiple dynamic Service Groups, identified by a SERVICE-NUMBER, can be used. However, the configuration of the dynamic Service Groups is dictated by the CE Cluster.

### Multicast for WCCP

For WCCP v2, the router can listen to a multicast address for WCCP messages from the CEs. First you must enable multicast on the router and on the interface, then set the multicast address for the service and set the interface to listen for that address. When configured with the multicast address, the router will also reply at the multicast address. Otherwise it will reply at unicast.

```
R(config)# ip wccp {web-cache|SERVICE-NUMBER} group-address MULTICAST-IP
R(config-if)# ip wccp {web-cache|SERVICE-NUMBER} group-listen
```

### Mode: open vs closed

By default, WCCP servics run in open mode. That means that in the absence of a any Content Engines, the traffic is sent to the original destination. However, you can have the router drop the traffic if there is no Content Engine available if you configure the service to work in the closed mode:

```
R(config)# ip wccp {web-cache | SERVICE-NUMBER} mode {open|closed}
!default: open
```

For closed services only, you can define a service-list that defines the application (protocol type or port number). This service-list must match the definition received from the CE Cluster, otherwise the service won’t start.

```
R(config)# ip wccp {web-cache | SERVICE-NUMBER} service-list ACL mode closed
! ACL must match a protocol type or port number
```

### Filtering servers or traffic

For each service you can define what traffic should be redirected. Traffic permitted by the ACL will be redirected. In the absence of the ACL, all traffic is redirected.

```
R(config)# ip wccp {web-cache | SERVICE-NUMBER} redirect-list ACL
```

To limit the Content Engines that can register for a service, use:

```
R(config)# ip wccp {web-cache | SERVICE-NUMBER} group-list ACL
```

### Authentication

In version 2 you can also enable MD5 authentication for content engines using:

```
R(config)# ip wccp {web-cache | SERVICE-NUMBER} password PASSWORD
```

## Enable redirection on the interface

You can enable redirection for incoming or outgoing packets, using:

```
R(config)# ip wccp {web-cache | SERVICE-NUMBER} redirect {in|out}
```

### Check Outbound ACL

When incoming traffic is redirected, this operation is done after the input ACL was checked. However, when outgoing traffic is redirected, this operation is done before the output ACL was checked. See [Routing Order of Operations](https://nyquist.eu/routing-order-of-operations/) for details. This means that traffic that would normally be dropped is redirected before it gets to be checked by the outgoing ACL. You can force the router to check the outgoing ACL before redirecting if you configure globally:

```
R(config)# ip wccp outbound-acl-check
! or 
R(config)# ip wccp check acl outbound
```

### Exclude inbound interfaceL

Alos, when outgoing traffic is redirected, all incoming traffic is checked by the router as subject of redirection. You can define interfaces to skip this check if you want incoming traffic on them to be excluded from being redirected with:\
R(config-if)# ip wccp redirect exclude in\
Since traffic redirecting is done before NAT (see [Routing Order of Operations](https://nyquist.eu/routing-order-of-operations/)), it will completely bypass it. You can use the above command to exclude traffic from being checked for redirection and let it pass through NAT.

### Multiple services on an interface

When multiple services are configured on an interface, they are used in the order of their priority, as defined by the CE Cluster.\
If these groups are configured with a **redirect-acl**, then the traffic will only be redirected if permitted by the ACL. Traffic denied by the ACL will not be redirected. This means that only the best priority service-group is used for redirection, while the other service-groups are not skipped.

To enable the router to check the denied traffic agains the next priority service group, use the global config:

```
R(config)# ip wccp check services all
```
