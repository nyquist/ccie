# MPLS 101

## Labels

MPLS (Multi Protocol Label Switching) adds a 4 Byte header between the Layer 2 and Layer 3 header for each label. The header has 4 fields:

* 20 bits – Label – locally significant on the link (similar to Frame Relay DLCI)
* 3 bits – EXP – used for QoS markings
* 1 bit – Bottom of Stack – signals the last label
* 8 bits – TTL

Certain applications of MPLS require the use of multiple labels. This is achieved by adding multiple MPLS headers before the L3 header. This operation is called “label stacking”. The last label has the Bottom of Stack (BoS) bit set to 1. All others have it set to 0.

Routers participating in MPLS operations are called LSR (Labels Switch Routers). Based on their position in the traffic path, they act as Ingress LSR (receives unlabeled packets and sends labeled packets), Intermediate LSR(receives labeled packets ands send labeled packets – but the label was swapped, as it has only local significance) and Egress LSR (receives labeled packets and send unlabeled packets).

MPLS labels are bound to FECs (Forwarding Equivalency Classes). All packets inside a FEC follow the same route to the next hop, so they are marked with the same label. In a network that doesn’t use traffic engineering, each IGP destination (including static and connected) should represent a FEC.\
The operations allowed on labels are:

* Push – replace the top label with another and add a new label on top.
* Swap – replace the top label with another label
* Pop – remove the top label

### Enable MPLS per interface

CEF must be on:

```
R(config)# ip cef
```

Then, start MPLS on each interface:

```
R(config-if)# mpls ip
```

This command will automatically start LDP or TDP based on the following setting:

```
! Global:
R(config)# mpls label protocol {ldp|tdp}
! Per interface:
R(config-if)# mpls label protocol {ldp|tdp|both}
```

To verify, use:

```
R# show mpls interfaces
R# show mpls ldp neighbors
```

### TTL

By default, the IP TTL field is copied into the MPLS TTL and decremented at each hop. This will make the MPLS cloud visible in traceroutes. To make it invisible:

```
R(config)# no mpls ip propagate-ttl
```

When the packet reaches the Egress router, the MPLS TTL value is copied into the IP TTL field (except when the MPLS TTL value is higher than the existing IP TTL value to prevent loops)\
In a label stack, an LSR only modifies the top label and doesn’t modify the other labels below it.\
When the TTL reaches 1, the TTL expires and the packet is dropped. In classic IP networks, an ICMP “Time exceeded” message would be sent back to the original sender. In MPLS, it can happen that the router that dropped the packet doesn’t know how to reach the source of the packet. To address this, the router sends the ICMP message towards the destination, in order to find the Egress router. The Egress router should know how to reach the source of the dropped packet and it will now send the ICMP packet on the path to the source. (it may very well go through the router that dropped the packet initially).

## Label Distribution

Label bindings can be set static or dynamic. For dynamic distribution of labels between LSRs, you can use:

* TDP (Tag Distribution Protocol – Cisco prioprietary) – similar to LDP
* LDP (Label Distribution Protocol)
* RSVP (Resource reSerVation Protocol) – used for MPLS Traffic Engineering
* MP-BPG(Multi Protocol BGP)

### LDP

#### **LDP Neighbor Adjacency**

LDP uses UDP 646 to 224.0.0.2 for neighbor discovery. Once discovered, it will create a Neighbor Adjacency using TCP 646 to remote LDP Router ID using its own Router-ID as the source. If you don’t want to use the Router-ID as the source, use:

```
R(config-if)# mpls ldp discovery transport-address {IP-ADDR | INTERFACE}
```

The Router ID is detetrmined just like in OSPF and it can be overridden using:

```
R(config)# mpls ldp router-id {IP-ADDR|INTERFACE}
```

To verify LDP adjacencies, use:

```
R# show mpls ldp neighbors
```

#### **Label bindings**

Each router that runs MPLS, will assign labels to all routes in its routing table( connected interfaces and IGP learned routes). These are called local bindings. Usually label allocation start with 16 and move forward. You can see the label allocation space with the command:

```
R#show mpls label range
Downstream Generic label region: Min/Max label: 16/100000
```

Via LDP, or another label distribution protocol, it will advertise the local bindings to its neighbors and will also receive bindings advertised by its peers. These are called remote bindings. The bindings make up the LIB (Label Information Base). You can see it using:

```
R# show mpls ldp bindings [DESTINATION]
!E.g:
R# show mpls ldp bindings 10.9.9.0 24
  tib entry: 10.9.9.0/24, rev 30
       local binding:  tag: 25
        remote binding: tsr: 3.3.3.3:0, tag: 47
        remote binding: tsr: 4.4.4.4:0, tag: 32
        remote binding: tsr: 5.5.5.5:0, tag: 64
```

If a LSR receives binding information for the same destination from multiple LSR neighbors, it will select the information coming from the next hop of that specific destination taken out of the routing table (RIB). The selected information will make up the LFIB (Label Forwarding Information Base) and it will include local label, outgoing operation (pop, swap), destination prefix, outgoing interface and next hop. You can check the LFIB with:

```
R# show mpls forwarding-table [DESTINATION]
!E.g:
R# show mpls forwarding-table 10.9.9.0 24
Local  Outgoing    Prefix            Bytes tag  Outgoing   Next Hop
tag    tag or VC   or Tunnel Id      switched   interface
25     32          10.9.9.0/24       0          Fa0/1     3.3.3.3
26     45          10.8.8.0/24       0          Se0/1/0   point2point
27     Pop Tag     10.7.7.0/24       0          Fa0/1     3.3.3.3
28     Untagged    10.6.6.0/24       0          Fa0/2     6.6.6.6
```

Pay attention to entries that only have a local or a remote binding. This might indicate a problem with the IGP. If an entry only has local entry, it means no other neighbor knows about this prefix. Similarly, if only remote entries appear, then this router doesn’t know how to reach the prefix advertised by one of its LDP routers. Routes that are directly connected are advertised with a NULL label, to let the penultimate hop know that the next router will forward the packet based on the IP address. See [PHP](broken-reference) for details.

The operation field in the LFIB will show:

* **Pop tag** – if the top label will be popped. Usually when [PHP](broken-reference) is in action
* **LABEL** – for the SWAP operation – Usually for all destinations that are more than 1 hop away
* **Untagged** – when packets are forwarded as IP – for all destinations that point to non MPLS neighbors

For ingress routers, forwarding is done based on CEF entries, and you can see there how a label is imposed (PUSH):

```
R# show ip cef [DESTINATION] 
```

#### **PHP – Penultimate Hop Popping**

PHP is an MPLS feature which says that if a packet must be forwarded to the last MPLS enabled router in the path (the PE), the MPLS label should be popped, and the packet should be sent as IP. The reason for this is that if we would send the packet labeled, the egress router should perform both an MPLS lookup based on the label and an IP lookup based on the destination IP. The routers advertise the destinations that do not run MPLS with an “Implicit NULL” label (Label value: 3). This way, the penultimate router knows that it should POP labels when it is sending for those destinations. This is the default behavior of Cisco IOS routers.

With that in mind, remember that there is an EXP field in the MPLS header that is used for QoS markings. If PHP is in place, the MPLS header is removed and QoS information is lost when the packet reaches the Egress router. To avoid these situations, the egress router advertise the destinations with an “Explicit NULL” label (Label value: 0 for IPv4, 2 for IPv6). This will make the penultimate router to send the packets for that prefix with an MPLS header which includes the QoS markings in the EXP field and NULL in the Label field. To configure an egress router to advertise Explicit NULL labels, use:

```
R(config)# mpls ldp explicit-null [for PREFIX-ACL] [to PEER-ACL]
```

#### **Limit LDP Advertisements**

By default, LDP will advertise labels for all entries in the routing table. You can disable this, by using:

```
R(config)#no mpls ldp advertise-labels
```

and then use a standard ACL to advertise only specific routes:

```
R(config)# mpls ldp advertise-labels for ACL
```
