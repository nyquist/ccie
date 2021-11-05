# Inter Domain Multicast

## MSDP (Multicast Source Discovery Protocol)

PIM-DM is not a viable solution to use over Internet, so we will use PIM-SM for interdomain multicast routing. PIM-SM Requires a Randezvous Point. In PIM-SM, when a source starts to transmit, the DR on it’s segment will register the source with the RP.Then, the RP can join the SPT to the source. But what happens when you have the source in one AS and receiver in another one? You could have a single RP for both ASes, but this is a highly improbable since 2 ASes should be under different administration. The better solution is to use MSDP.\
MSDP, defined in [RFC 3618](https://www.ietf.org/rfc/rfc3618.txt), is used between different domains/ASes to signal each other about the sources they have in the domain. In fact, the RP in one AS will signal the RP in the other AS about a new source that wants to transmit. Knowing the source, the RP in the second AS can now join the SPT tree towards the source in the first domain.

MSDP is somewhat similar to BGP because the RP in each domain establishes TCP connections with the RP in the other domains. MSDP uses TCP 639 for the TCP connection.\
When a RP discovers a new source in its domain (via PIM REGISTER), it sends a SOURCE-ACTIVE (SA) message to all MSDP Peers.

When a RP has a (\*,G) entry for the group (meaning a receiver on the shared tree), it will JOIN the SPT tree to the source by sending an (S,G) JOINs.

You can see now that MSDP is used only for exchanging information about existing sources, not for the actual exchange of data.

If there are multiple interconnected domains, a router may receive MSDP information from multiple neighbors. It will perform a special RPF check by checking if the MSDP message that was received came on the same interface as the one that points to the next-hop for reaching the Originator ID, in the BGP table. If also the multicast address-family is used, the information in advertised here is used for the RPF check.

First you must define the MSDP peers:

```
R(config)# ip msdp peer PEER-IP [connect-source INTERFACE] [remomte-as AS-NUMBER]
! configure the MSDP peer and the source interface
```

Additional settings can be configured for each pear:

```
R(config)# ip msdp password peer PEER-IP PASSWORD
! form md5 authentication
R(config)# ip msdp sa-limit PEER-IP LIMIT
! Limit the number of SA messages that it can receive. Prevents DoS
R(config)# ip msdp keepalive PEER-IP KEEPALIVE HOLD-TIME
! Default: KEEPALIVE 60 , HOLD-TIME: 75
R(config)# ip msdp sa-filter {in|out} PEER-IP [list ACL|route-map ROUTE-MAP]
! Restrics the information that is advertised (out) to or is accepted (in) from other peers
! deny entries in the ACL/ROUTE-MAP will be filtered
```

or for the MSDP process:

```
R(config)# ip msdp originator-id INTERFACE
! changing the default Origiantor ID. By default it's the RP address
R(config)# ip msdp redistribute [list ACL]
! Restricts which sources are advertised to other peers
```

### Anycast RP

Anycast RP is an application of multicast inside a domain, that offers load sharing and backup for a RP. Two or more routers are configured with the same /32 Loopback address, and that address is advertised as the RP. The PIM SM routers in the domain will use the Anycast RP that is closest to them. Because the clients and the sources might end up using different RPs, a MSDP session is used to exchange source information between the RPs.\
Fist, define the 2 routers with the same Loopback IP address, and set this address as the PIM RP.\
Then, define the 2 routers as MSPD peers. Make sure you don’t use the same another address as the connect source:

```
R(config)# ip msdp peer PEER-IP [connect-source INTERFACE]
```

This configuration also provides redundancy for the RP.

## MP-BGP for multicast

Multicast only works if the 2 domains are interconnected with links that run PIM. Also, multicast traffic must pass RPF check in order to be forwarded. When performing the RPF check, a router that uses PIM will look into the unicast routing table for the route that points to the destination of the multicast traffic. The unicast routing table may be populated with routes from all routing protocols, including BGP.\
However, what if you need the inter-domain multicast traffic to use another link, different than unicast traffic? Well, you could use static mroutes, but these are local to each router. Anoher option is to use MP-BGP’s **address-family ipv4 multicast**\
What this does, is to exchange unicast routes between routers, but these routes will not be used for unicast routing, instead they will be used for RPF checks when routing multicast traffic.\
Actually, you can have the same routes that are advertised by unicast ipv4 BGP advertised in multicast ipv4 BGP. But since they are used for different functions, there’s no problem. Using this method you can apply different routing policies for multicast and unicast traffic.\
To enable the exchange of routes used for multicast routing, you must activate neighbors in the **address-family ipv4 multicast**. This address-family runs completely independent of the ipv4 unicast address-family, so you will need to configure it appropriately (route-reflectors, advertised networks, redistribution, and so on).

An MP-BGP multicast learned route is preferred over any unicast routes when performing the RPF check.

```
R(config-router)# address-family ipv4 multicast
R(config-router-af)# neighbor NEIGH-ADDR activate
! If needed, originate routes or do other settings in the address-family
R(config-router-af)# network NETWORK mask MASK
```

To check the status of MP-BGP for multicast, use:

```
R# show bgp ipv4 multicast [summary|neighbors NEIGH-ADDR]
```
