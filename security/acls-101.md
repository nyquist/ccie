# ACLs 101

An ACL contains one or more ACEs (Entries) that permit or deny traffic and have an implicit deny any at the end.

## Numbered ACLs

### Standard ACLs

```
R(config)# access-list ACL-NUMBER {permit|deny} {IP-ADDRESS [WILDCARD] | any} [log]
! ACL-NUMBER: 1-99, 1300-1999
! when the wildcard is missing, a default of 0.0.0.0 is considered
! any <=> IP-ADDRESS 255.255.255.255
! log = adds an entry in the log (one entry every 5 minutes)
R(config)# access-list ACL-NUMBER remark COMMENT
```

You cannot edit one individual entry in a numbered ACL. The ACL must be deleted and re-created.

### Extended ACLs

```
R(config)# access-list ACL-NUMBER {permit|deny} PROTOCOL {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} [OPTIONS] [log|log-input]
! ACL-NUMBER: 100-199, 2000-2699
! PROTOCOL = ip, tcp, upd, protocol number, etc
! any <=> IP 255.255.255.255
! host IP <=> IP 0.0.0.0
! log = adds an entry in the log (one entry every 5 minutes)
! log-input = adds additional info to the log (input interface, source MAC)
! OPTIONS: dscp, precedence, tos, IP Options, fragments, ttl...
R(config)# access-list ACL-NUMBER remark COMMENT
```

#### **Established**

One option for TCP traffic is to allow only packets that are part of an established connection. Packets that are part of an established TCP connection have the ACK or RST bit set. When using the **established** keyword at the end of TCP extended ACL, you can match only these packets:

```
R(config-ext-acl)# {permit|deny} tcp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} established
```

This is useful when you don’t want one side to initiate the connection, but you need it to be able to respond to connections initiated from the other side.

#### **Matching Tips**

Here are some tips for matching traffic with extended ACLs:

```
! Match RIP:
R(config-ext-acl)# {permit|deny} udp {any|SRC-IP SRC-WILDCARD} any eq {520|rip}
! Match EIGRP
R(config-ext-acl)# {permit|deny} eigrp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD}
! Match OSPF
R(config-ext-acl)# {permit|deny} ospf {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD}
! Match BGP
R(config-ext-acl)# {permit|deny} tcp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} eq {179|bgp}
! Match LDP
R(config-ext-acl)# {permit|deny} tcp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} eq 646
R(config-ext-acl)# {permit|deny} udp {any|SRC-IP SRC-WILDCARD} any eq 711
! Match TDP
! Match LDP
R(config-ext-acl)# {permit|deny} tcp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} eq 711
R(config-ext-acl)# {permit|deny} udp {any|SRC-IP SRC-WILDCARD} any eq 646
! Match FTP
R(config-ext-acl)# {permit|deny} tcp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} range 20 21
!Match ping
R(config-ext-acl)# {permit|deny} icmp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} echo
R(config-ext-acl)# {permit|deny} icmp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} echo-reply
!Match traceroute
R(config-ext-acl)# {permit|deny} icmp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} time-exceeded
R(config-ext-acl)# {permit|deny} icmp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} port-unreachable
R(config-ext-acl)# {permit|deny} udp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} range 33434 33464
! Match Path MTU Discovery
R(config-ext-acl)# {permit|deny} icmp {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} packet-too-big
```

To see a list of the most used well-known ports, use:

```
R# show ip port-map
```

## Named ACLs

### Standard ACLs

```
R(config)# ip access-list standard ACL-NAME
R(config-std-nacl)# {permit|deny} {IP-ADDRESS [WILDCARD] | any} [log]
R(config-std-nacl)# remark COMMENT
```

### Extended ACLs

```
R(config)# ip access-list extended ACL-NAME
R(config-ext-nacl)# [seq-number] {permit|deny} PROTOCOL {any|SRC-IP SRC-WILDCARD} {any|DST-IP DST-WILDCARD} [OPTIONS] [log|log-input]
!seq-number can be used to edit an ACL entry or to insert one entry into an existing ACL
!if seq-number is missing, the ACE will be appended with a seq-number value of max-seq-number+10
R(config-ext-nacl)# remark COMMENT
```

Since ACEs can have a custom seq-numbere, they can be re-sequenced to allow other insertions:

```
R(config)# ip access-list resequence ACL-NAME SEQ-START INCREMENT
```

## Using ACLs

### Filter traffic on an interface

```
R(config-if)# ip access-group ACL {in|out}
```

### Limit CLI access

```
R(config-line)# access-class ACL {in|out}
! ACL - can only be standard ACL
! in - ACL applies to inboud connections
! out - ACL applies to outbound connections. ACL matches destination address
```

## Fragments

By default, an ACL without the fragments keyword works in the following manner:

* If the ACL contains only Layer 3 information (SRC-IP, DEST-IP) the entry is applied to nonfragmented packets, initial fragments and non-initial fragments
* If the ACL contains both Layer 3 and Layer 4 information (SRC-IP, DEST-IP, Layer 4 protocol):
  * The entry is applied to nonfragmented packets and initial fragments
  * If the entry is a permit, it will be applied to non-initial fragments, but since the Layer 4 information is missing, it will only match on Layer 3 information, and will ignore Layer 4 information.
  * If the entry is a deny, it will be ignored for non-initial fragments, and the next ACE is processed

When using the **fragments** keyword, the entry is applied only to non-initial fragments. Non-fragmented packets and initial fragments are not matched. This keyword cannot be used on entries that use Layer 4 information.

## Logging ACLs

You must use the log keyword at the end of an ACE to enable generation of syslog messages when the ACE is hit. One message will be generated every 5 minutes.\
You can set the number of hits that will generate a log entry using:

```
R(config)# ip access-list log-update threshold HITS
```

For extended ACLs, you can use log-input keyword to add more information to the log (input interface, source MAC). You can add custom tags to the logs by using the log keyword followed by the tag, or you can generate automatic hashtags for each ACE by setting:

```
R(config)# ip access-list logging hash-generation
```

These hashtags will identify the ACE that generated them in the log output.
