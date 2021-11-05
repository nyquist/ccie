# Switch ACLs

## Port ACLs

Can only be applied on physical L2 interfaces on a switch (not on etherchannels). They can only be applied on the inbound direction.\
A port ACL can be either a Standard ACL, an Extended ACL or an Extended MAC ACL. Only one standard or extended ACL and one Extended MAC ACL can be applied on a port. It will filter packets regardless of the VLAN they come on.\
To configure a MAC Extended ACL, use:

```
Sw(config)# mac access-list extended MAC-ACL-NAME
Sw(config-ext-macl)# {deny|permit} SRC-MAC MAC-MASK DST-MAC DST-MASK [ETHERTYTPE ETHERTYPE-MASK]
! You can Replace MAC MASK with any or host MAC
! ETHERTYPE identifies the type of Layer 2 frame
! No IP traffic is matched by MAC ACLs
```

To apply a MAC ACL to a port, use:

```
Sw(config-if)# mac access-group {MAC-ACL-NAME} in
```

To apply an IP ACL to a port, use:

```
Sw(config-if)# ip access-group {ACL-NUMBER|ACL-NAME} {in|out}
```

## Router ACLs

A router ACL can be applied on L3 SVIs, and can be applied in both inbound and outbound directions. Only one Standard ACL or Extended ACL is allowed on each direction\
To set a rotuer ACL, use:

```
Sw(config-if)# ip access-group {ACL-NUMBER|ACL-NAME} {in|out}
```

## VLAN Maps

A VLAN Map can filter all traffic inside a VLAN, regardless of the direction. But then, things get complicated. VLAN maps can filter traffic IP traffic (based on L3/L4 information) and non-IP traffic (based on L2 information – MAC Addresses). A VLAN Map can have several entries (identified by SEQ-NUMBER) and when a packet is checked, it is checked against the match commands of the entries, in the order of SEQ-NUMBER until the first match occurs. When the match occurs, it will drop or forward the packet based on the information in that entry. If there is no match, but there are entries for that kind of traffic (IP or non-IP) then the packet is dropped. If there is no match, but there are no entries for that kind of traffic (IP or non-IP), then the packet is forwarded. To be more preceise:

* An IP packet enters a VLAN
  * Are there any entries that **match ip**?
  *
    * No. The packet is forwarded
    * Yes. Are there any entries that match the packet?
      * Yes. Do what the entry says (drop or forward)
      * No. Drop the packet
* A non-IP packet enters a VLAN
  * Are there any entries that **match mac**?
  *
    * No. The packet is forwarded
    * Yes. Are there any entries that match the packet?
      * Yes. Do what the entry says (drop or forward)
      * No. Drop the packet

To configure a VLAN Map, follow these steps:

```
Sw(config)# vlan access-map VLAN-MAP-NAME [SEQ-NUMBER]
Sw(config-access-map)# action {drop|forward}
Sw(config-access-map)# match {ip address IP-ACL | mac address MAC-ACL-NAME}
Sw(config-access-map)# exit
Sw(config)# vlan filter VLAN-MAP-NAME vlan-list VLAN-LIST
```
