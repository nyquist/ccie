# DHCP 101

## DHCP Server

### DHCP Pools

On a router, you have to create one or more pools of DHCP addresses available for lease. When a DHCP server receives a DHCP request, it will know what pool to use based on the IP address of interface that received it. If the request came from a host that is not on that subnet, the router will try to match the GIADDR field which contains the IP address of the interface that received the DHCP request on the relay server.\
To define a pool, use:

```
R(config)# ip dhcp pool POOL
```

Now you can define the pool attributes:

```
R(dhcp-config)# network NETWORK-ADDR [NETMASK]
R(dhcp-config)# domain-name DOMAIN
R(dhcp-config)# dns-server SERVER1 [SERVER2 ...]
R(dhcp-config)# default-router GATEWAY1 [GATEWAY2 ...]
R(dhcp-config)# option CODE [instance NUMBER] {ascii STRING | hex STRING | IP-ADDR}
R(dhcp-config)# lease {DAYS [HOUR [MINUTES]]|infinte}
! default - 1 DAY
R(dhcp-config)# update {arp|dns}
! updates local arp and dns info with data from dhcp server.
! update arp is used for Authorized ARP - See ARP 101 article
```

You can exclude some IP’s from being used in the DHCP pool using:

```
R(config)# ip dhcp excluded-address START-IP [END-IP]
```

### Manual Bindings

To make manual bindings you have to create additional pools, one for each static binding. In these pools you must specify the host IP-Address and an identifier for the host – Client ID or Hardware Address. Clients are matched by the hardware address only if they don’t send a client ID. Cisco routers always send a Client ID!

```
R(config)# ip dhcp pool STATIC-HOST-1
R(dhcp-config)# host IP-ADDR [NETMASK]
R(dhcp-config)# client-identifier CLIENT-ID
R(dhcp-config)# hardware-address MAC-ADDRESS [PROTOCOL-TYPE|HARDWARE-NUMBER]
R(dhcp-config)# client-name HOSTNAME
```

Defining one POOL for each client can become an administrative nightmare. Another option is to use static mappings from a file similar to the one that the router saves when using the database agent.\
To enable the use of a static file, use this command in a Network Pool:

```
R(dhcp-config)# origin file URL
```

### ODAP – On Demand Address Pool

A pool can be configure to use one or more subnets from another DHCP or AAA server. This is usually used by providers to assign addresses from the same pool to different customers. To define an ODAP pool, use:

```
R(config)# ip dhcp pool ODAP-POOL
R(dhcp-config)# origin {dhcp|aaa|ipcp}
! Optionally the pool can be assigned to a vrf
R(dhcp-config)# vrf VRF-NAME
! Now you can import the dhcp options from the server into the current pool
R(dhcp-config)# import all
```

The ODAP server must also be configured to reply with the requested subnets. To do this, configure the server pool with:

```
R(config)# ip dhcp pool SERVER-POOL
R(dhcp-config)# network NETWORK-ADDR NETMASK
R(dchp-config)# subnet prefix LEN
! Defines size of the subnet that can be allocated to ODAP clients
```

### DHCP Relay Agent

By default, the list of DHCP bindings is kept in memory by the router and they are lost once the router reloads. You can enable the router to use a database agent, that is a location where the router can save the list of the bindings. This file can also be used to recover the list in case of reloads. To enable this feature, use:

```
R(config)# ip dhcp database URL
! URL can be a local or a remoate location. Ex: flash:dhcp.txt
```

### DHCP Classes for Option 82

When a host requests an address via DHCP, the server uses the incoming interface IP or the GIADDR field in the DHCP request (filled by a relay agent with it’s incoming interface IP) that would match the network address defined for a pool. Additionally, it can use the client-id or hardware-address for specific manual bindings.\
Option 82 is a special field in a DHCP packet that can contain additional information that may identify a host. Switches can be configured to insert information in this field (and in the Relay Information Option field). See [DHCP Snooping](https://nyquist.eu/dhcp-snooping-and-dai/). To process this information, the DHCP server on a router needs to use DHCP classes. By default, DHCP classes are on, but if disabled, they can be enabled again with:

```
R(config)# ip dhcp use class
```

When you configure a class, you actually define the option fields that the router will compare when it receives a DHCP request. The match is done bitwise.

```
R(config)# ip dhcp class CLASS-NAME
! Define option fields
R(config-dhcp-class)# option OPTION-NUMBER hex OPTION-HEX-VALUE [mask BITMASK]
! Add Relay Agent Information fields:
R(config-dhcp-class)# relay agent information
R(config-dhcp-class-relayinfo)# relay-information hex RELAY-HEX-VALUE [mask BITMASK]
! If missing, it matches anything
```

Then apply the class to an existing DHCP pool. When you do this, the pool will be used for allocation only if at least one class is matched by a DHCP request.

```
R(dhcp-config)# class CLASS-NAME
! You can assign a sub-range of the pool to this class:
R(config-dhcp-pool-class)# address range START END
! Or forward the requests to another DHCP server:
R(config-dhcp-pool-class)# relay target OTHER-DHCP-SERVER
```

The value of the Option82 field is also added back to the DHCP reply messages that the server sends to its clients.

If the relay agent inserts Option 82 but doesn’t add GIADDR field, the router will drop the DHCP message unless you configure it to trust such messages:

```
! Globally, for all interfaces:
R(config)# ip dhcp relay information trust-all
! or just for one interface:
R(config-if)# ip dhcp relay information trusted
```

Another option is to disable the insertion of Option82 on the switch:

```
Sw(config)# no ip dhcp snooping information option 
```

## DHCP Relay Agent

You can forward DHCP requests to another server using the feature of [forwarding UDP protocols](https://nyquist.eu/multicast-101/#61\_Convert\_broadcast\_to\_unicast\_8211\_Helper\_Address). For DHCP, the following command should be enough:

```
R(config-if)#ip helper-address REMOTE-SERVER
```

The requests will be sent as unicast and the relay agent will add the GIADDR information (IP address of the interface that received the DHCP request)\
Another option is to make the router forward DHCP requests from within a pool:

```
R(config)# ip dhcp pool POOL-NAME
! The relay source will match be used to match the incoming interface or GIADDR
R(dchp-config)# relay source NETWORK-ADDR MASK
! The relay destination will define where the DCHP request is forwarded
R(dhcp-config)# relay destination REMOTE-SERVER
```

When using classes, you can define a relay target for each class:

```
R(config-dhcp-pool-class)# relay target OTHER-DHCP-SERVER
```

### Option 82

The DHCP relay can be configured to add Option82 information to the DHCP requests that it forwards. You can enable this globally or per interface:

```
R(config)# ip dhcp information option
Rack1R4(config-if)#ip dhcp relay information option-insert [none]
```

Also by default, the router will also check the reply messages from the server before forwarding them to the host. If they don’t have the Option82 information echoed back in the reply packet, it will be dropped. You can disable this check globally, or per interface:

```
R(config)# no ip dhcp relay information check
R(config-if)# ip dhcp relay information check-reply [none]
! Use none to disable
```

A Cisco switch can insert Option82 information into a DHCP request. When it receives DHCP requests that already have this information attached, the router will replace it with its own. You can configure how it should treat these packets with any of the following commands:

```
R(config)# ip dhcp relay information policy {drop|keep|replace}
R(config-if)# ip dhcp relay information policy-action {drop|keep|replace}
```

Again, if the relay agent inserts Option 82 but doesn’t add GIADDR field, the router will drop the DHCP message unless you configure it to trust such messages:

```
! Globally, for all interfaces:
R(config)# ip dhcp relay information trust-all
! or just for one interface:
R(config-if)# ip dhcp relay information trusted
```

### 3DHCP Client

Settings for DHCP clients can be configured globally, or on each interface:

```
! Global DHCP Client config
R(config)# ip dhcp-client ?
  broadcast-flag     Set the broadcast flag
  default-router     Set DHCP default router related information
  forcerenew         Enable forcerenew client processing
  network-discovery  Configure parameters for network discovery
  update             Configure automatic updates
```

```
! Interface DHCP Client config
R(config-if)# ip dhcp client ?
  class-id   Specify Class-ID to use
  client-id  Specify Client-ID to use
  hostname   Specify hostname to use
  lease      Requested address lease time
  mobile     Mobile client configuration parameters
  request    Specify options (not) to request
  route      Options for routes installed by dhcp
  update     Dynamically update information
```

Another option is to specify the information when you enable the DHCP client:

```
R(config-if)# ip address dhcp [client-id INTERFACE][hostname NAME]
```

A DHCP router will always include a client-id in it’s request. This means that a Cisco DHCP server will not use it’s mac-address when searching for a host pool used for address assignment.\
You can see what is the value of the client-id sent by the router in it’s DHCP requests with:

```
R#sh dhcp lease
```

If you want to match this value on the DHCP server, you will have to use the hex-value but in a dotted format. So a better option would be to change the client id before sending the request.\
To see debug information for the client, use:

```
R# debug dhcp
```

You can use 2 exec commands to release or renew the DHCP address:

```
R# release dhcp INTERFACE
R# renew dhcp INTERFACE
```
