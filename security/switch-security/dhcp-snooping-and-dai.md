# DHCP Snooping and DAI

## DHCP Snooping

DHCP snooping can prevent unauthorized DHCP servers to reply to DHCP requests. A switch can define interfaces as trusted or untrusted. A trusted interface is where a DHCP server should be connected. On such interfaces, DHCP server messages are allowed. On all other untrusted ports, DHCP server messages are droped. Also, while this feature runs, the switch builds a DHCP binding database, where it maps all MAC addresses to the IP addresses they received via DHCP.\
To enable DHCP snooping, use:

```
! 1. Enable DHCP Snooping globally
Sw(config)# ip dhcp snooping
! 2. Then enable DHCP snooping on the VLAN
Sw(config)# ip dhcp snooping vlan VLAN-RANGE
! 3. Set up the interfaces as trusted. By default they are untrusted
Sw(config)# interface INTERFACE
Sw(config-if)# ip dhcp snooping trust
```

### Option 82

DHCP Option 82 allows a DHCP server to identify a host by the port on the switch it connects to, in addition to the host MAC Address.\
To enable the switch to add the Option 82 field to the DHCP request message, use:

```
Sw(config)# ip dhcp snooping information option
```

You can also limit the number of DHCP packets that are received on an interface, using:

```
Sw(config)# ip dhcp snooping limit raate PPS
```

To modify the default information that is added, use:

```
Sw(config)# ip dhcp snooping information option format remote-id [string STR|HOSTNAME]
```

By default, when DHCP snooping is on, a switch will drop DHCP requests with Option 82. The switch considers that only it can add this information and it considers those requests as illegitimate.\
This is a good idea on access switches, but on an aggregation switch, it might end up in dropping DHCP requests that had their option 82 inserted by legitimate access switches. To permit such requests, use:

```
Sw(config)# ip dhcp snooping information option allow-untrusted
```

DHCP Snooping is known to add empty giaddr field in the DHCP messages, which will make most DHCP servers ignore them. There are 2 solutions:\
1\. make the server trust DHCP messages with empty giaddr field

```
R(config)# ip dhcp relay information trust-all
```

2\. disable option 82 insertion on the switches

```
Sw(config)# no ip dhcp snooping information option 
```

### DHCP Snooping Binding Database

Information regarding the legitimate hosts on the network are stored in the DHCP Snooping Biding Database. This database contains the MAC address and the IP address that was offered in the DHCP process. This database is lost upon restart. To prevent this, you can specify a DHCP Snooping Binding Database Agent, after which you can set a location where to save this database:

```
Sw(config)# ip dhcp snooping database PROTOCOL://FISER
```

You can also add static information to the database, using:

```
Sw(config)# ip dhcp snooping binding MAC vlan VLAN-ID inteface INTERFACE expiry SEC
```

To verify, use:

```
Sw# show ip dhcp snooping database [detail]
```

## IP Source Guard

You can limit traffic on an untrusted port to a single source IP or MAC address, the ones that are found in the Snooping Binding Database for the specified port. A port ACL is applied to the interface in order to block all other traffic. This ACL will take precedence over any other router ACL or VLAN map that might affect the port.\
To enable IP Source Guard, use one of the following:

```
Sw(config-if)# ip verify source
!Enables IP address filtering
Sw(config-if)# ip verify source port-security
!Enables both IP and MAC address filtering
```

If using the port-security option, the MAC address in the DHCP packet is not learned as a secure address. It will be learned only when it starts to send non-DHCP traffic.\
To verify, use:

```
Sw# show ip source binding [IP-ADDR] [MAC] [dhcp-snooping|static] [interface] INTERFACE [vlan VLAN-ID]
```

If DHCP is not an option, or there are static hosts, you can enable IP Source Guard even for them, using:

```
Sw(config-if)# ip verify source tracking port-security
```

This will track the first non-DHCP MAC it receives and will only let it send and receive packets.

## Dynamic ARP Inspection (DAI)

DAI intercept ARP Replies and will drop those for which the MAC-IP binding is not found in the DHCP Snooping Binding Database. This means that you need to enable DHCP Snooping in order to run Dynamic ARP Inspection.\
To enable DAI, use:

```
Sw(config)# ip arp inspection vlan VLAN-RANGE
```

DAI intercepects ARP replies only on untrusted interfaces. To set an interfaces as trusted, use:

```
Sw(config-if)# ip arp inspection trust
! Usually only host interfaces are set as trusted
```

To verify, use:

```
Sw# show ip arp inspection {interfaces | [statistics] vlan VLAN-RANGE}
```

You can set a limit for the number of ARP requests that you receive on a port:

```
Sw(config)# ip arp inspection limit {rate PPS [burst interval SEC]|none}
!Default: 15 PPS for untrusted interfaces
```

If the rate goes over the limit, the port is errdisabled. To auto enable it, use:

```
Sw(config)# errdisable recovery cause arp-inspection interval INTERVAL
```

### ARP ACLs

When DHCP is not available, you can still enable ARP inspection using ARP ACLs.\
To set up an ARP ACL, first define it:

```
Sw(config)#arp access-lists ACL-NAME
Sw(config-arpacl)# permit ip host SRC-IP mac host SRC-MAC [log]
```

Then apply the ACL on a VLAN:

```
Sw(config)# ip arp inspection filter ACL-NAME vlan VLAN-RANGE[static]
! static - adds an explicit deny to the end of the ARP ACL-NAME
```

### Extra Validations

Apart from discarding ARP packets with invalid IP-to-MAC bindings, DAI can perform additional validation checks on ARP packets:

```
Sw(config)# ip arp inspection validate {[src-mac]|[dst-mac][ip]}
! src-mac: checks the source MAC address in the Ethernet header against the sender MAC address in the ARP body
! dst-mac: check the destination MAC address in the Ethernet header against the target MAC address in ARP body
! ip: checks the ARP body for invalid IP addresses, like 0.0.0.0 or 255.255.255.255
```
