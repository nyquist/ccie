# Cisco IOS Firewall

## CBAC – Context Based Access Control

CBAC allows examination of traffic at the Application Layer, not just Layer 3 or Layer 4 as in ACLs. It can maintain session information and create temporary openings to allow return traffic for permissible sessions.\
CBAC maintains a state table both for TCP and UDP (aproximated state – since the service is connectionless). Packets in the return traffic must match the information in the state table to be allowed.\
CBAC is set on an interface in one direction (incoming or outgoing) and will only inspect those packets that passed the ACL in that direction. (see [Order of operations](https://nyquist.eu/routing-order-of-operations/)).\
when traffic originates in the direction that the inspection rule is applied, it will generate shortcuts that will bypass any ACLs in the opposite direction, regardless of the interface. This means an inspect on the incoming traffic of the inside interface will allow returning packets on the outside (even if there is an ACL that denies them). Also, if the inspection is configured on the outside interface, in the outgoing direction, it will still allow return traffic.

To define the CBAC inspection rule, use:

```
R(config)# ip inspect name INSPECTION-NAME PROTOCOL [alert {on|off} [audit-trail {on|off}] [timeout SEC]
! alert: generates syslog messages
! audit-trail: generates more verbose messages
```

Apply the inspection rule on an interface:

```
R(config-if)# ip inspect INSPECTION-NAME
```

## TCP Intercept

TCP intercept is used to prevent servers from TCP SYN-flood attacks. When this type of attacks occur, an attacker sends multiple TCP SYN packets to a server, which should try to respond with a SYN-ACK and then keep this state information until the sender responds with an ACK and the three-way-handshake is completed. When the number of SYN packets received is high enough, the server may start dropping legitimate connections. The router can be configured to prevent this using the TCP Intercept feature.\
There are two modes of operation: active and passive. In the default active mode (intercept), the router will intercept the TCP SYN packets, and respond with a SYN-ACK on behalf of the server. Only when it receives an ACK back, it will create a three-way-handshake with the server and connect the 2 sessions with each other, thus stopping excess SYNs from reaching the server.\
In the passive mode(watch), the router forwards the SYN to the server but waits a limited amount of time (default:30 sec) for the three-way-handshake to complete, before it sends a RESET to the server to clear the connection.

```
! Traffic passing the following ACL will be intercepted.
! Usually matches the destination server
! Could be used not to intercept some known sources
R(config)# ip tcp intercept list ACL
! Define intercept mode: active (intercept) or passive (watch)
R(config)# ip tcp intercept mode {intercept | watch}
! Define timers
R(config)# ip tcp intercept {watch-timeout| finrst-timeout | connection-timeout} SEC
```

The TCP Intercept also has an aggressive mode in which any new connection will generate the drop of an old connection. This aggressive mode is automatically enabled based on a couple of thresholds regarding total number of incomplete connections or number of connections in the last minute.

```
R(config)# ip tcp intercept max-incomplete low LOW-VAL high HIGH-VAL
R(config)# ip tcp intercept one-minute low LOW-VAL high HIGH-VAL
R(config)# ip tcp interce drop-mode {oldest | random} 
```

## Unicast RPF (Reverse Path Forwarding)

Unicast RPF uses the same mechanism as the [Multicast RPF used by PIM](https://nyquist.eu/multicast-101/#2\_Reverse\_Path\_Forwarding) to verify that a packet arrived on the interface with the best route pointing to the source. If it did, then the packet is processed, if it didn’t, then it is dropped.

```
! CEF must be enabled
R(config)# ip cef
R(config)# interface INTERFACE
R(config-if)# ip verify unicast reverse-path [list ACL]
! spoofed packets that are permited by ACL are permited but can be logged
! spoofed packets that are denied by the ACL are dropped but can be logged
```

For multihomed environments, a loose version of uRPF can be enabled, where the packet does not need to enter a specific interface. A route to the source must exist in the routing table, otherwise the packet will be dropped. To enable the loose-mode uRPF, use:

```
R(config-if)# ip verify unicast source reachable-via any
```
