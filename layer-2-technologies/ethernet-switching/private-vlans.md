# Private VLANs

Private VLANs partitions a regular VLAN domain into subdomains. Such a subdomain is created when a primary VLAN is paired with a secondary VLAN. Only a switch in VTP Transparent mode supports Private VLANs

## Primary VLAN

To set a VLAN as Primary VLAN, use:

```
Sw(config)#vlan VLAN-ID
Sw(config-vlan)#private-vlan primary
```

After the secondary VLANs are configured, they are associated with the primary VLAN, using the following confing form withing the primary vlan:

```
Sw(config)#private-vlan association [add|remove] SECONDARY-VLAN-LIST
```

To verify, use:

```
Sw# show vlan private-vlan [type]
```

## Secondary VLANs

Secondary VLANs can be condfigured as Isolated or as Community VLANs. Private VLANs work over different switches, as long as the Private VLANs and the primary VLANs are carried over the trunk links.

### Isolated VLANs

Ports within the isolated VLAN cannot communicate with each other at Layer.

```
Sw(config-vlan)# private-vlan isolated
```

### Community VLANs

Ports within the Community VLAN can communicate with ports in the same Community VLAN but not with ports in other Community VLANs or with ports in the Isolated VLAN.

```
Sw(config-vlan)# private-vlan community
```

## Private VLAN Ports

To configure a port as part of a private VLAN, use:

### Promiscuous Ports

A promiscuous ports belongs to the primary VLAN and can communicate with all interfaces in the primary VLAN, including ports in the isolated and community secondary VLANs. To set up a promiscuous port, use:

```
Sw(config-if)# switchport mode private-vlan promiscuous
Sw(config-if)# switchport private-vlan mapping PRI-VLAN-ID {add|remove} SEC-VLAN-LIST
!The promiscuous port must be mapped to one or more Secondary VLANs
```

You can also map the VLANs to a promiscuous port using:

```
Sw(config-if)# switchport private-vlan association mapping PRI-VLAN-ID {add|remove} SEC-VLAN-LIST
```

### Isolated Ports

It is a port that belongs to the Isolted Secondary VLAN. It can only communicate with promiscuous ports.To set an isolated port, use:

```
Sw(config-if)# switchport mode private-vlan host
Sw(config-if)# switchport private-vlan host-association PRI-VLAN-ID SEC-VLAN-ID
!SEC-VLAN-ID must be an isolated VLAN
```

You can alos map the VLANs to a isolated port, using:

```
Sw(config-if)# switchport private-vlan association host PRI-VLAN-ID SEC-VLAN-ID
```

### Community Ports

It is a port that is part of a Community Secondary VLAN and it can only communicate with other ports in the same Community or with promiscuous ports.To set a community port, use:

```
Sw(config-if)# switchport mode private-vlan host
Sw(config-if)# switchport private-vlan host-association PRI-VLAN-ID SEC-VLAN-ID
!SEC-VLAN-ID must be a community VLAN
```

You can alos map the VLANs to a community port, using:

```
Sw(config-if)# switchport private-vlan association host PRI-VLAN-ID SEC-VLAN-ID
```

## Mapping Secondary VLANs to Primary Layer3 VLANs

To allow inter-vlan routing, the secondary VLANs must be mapped to the L3 SVI:

```
Sw(config)# interface vlan PRI-VLAN-ID
Sw(config-if)# private-vlan mapping [add|remove] SEC-VLAN-LIST
```

To monitor, use:

```
Sw# show interface private-vlan mapping
```
