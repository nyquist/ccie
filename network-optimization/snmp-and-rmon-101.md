# SNMP and RMON 101

## SNMP Basics

SNMP requires a SNMP Manager (usually a NMS – Network Management System) that sends SNMP requests (get or set) and a SNMP Agent (usually running on the managed device – router, switch) that responds to these requests. The SNMP Agent listens for requests on UDP port 161.

In addition, SNMP Agents can send unsolicited messages to the NMS when a certain event occurs. These messages are called Traps or Informs. The SNMP Manager listens for Traps or Informs on UDP port 162. Informs are similar to Traps but the SNMP Agent will resend them until they are acknowledged by the SNMP Manager. Traps are sent only once and no acknowledgement is expected.

Data that is available for the SNMP Agent is organized in a tree format known as a MIB (Management Information Base). The leaves of the MIB tree are known as OIDs (Object Identifiers) and they contain the actual data. A SNMP Manager can receive the data in one OID (get command) or it can navigate the tree by running multiple getNext commands. When receiving a getNext command, the SNMP agent recursively moves to the next OID and replies with its data. Support for the getBulk command was introduced in SNMPv2. It uses only one command to get data for multiple OIDs.

A Cisco Router supports SNMP v1, v2c and v3. SNMP v1 and v2c both use a community based security model, but SNMP v2 adds support for additional SNMP commands like getBulk (instead of multiple getNext commands)\
SNMPv3 only improves the security of SNMP by adding integrity checks, authentication and encryption.

## SNMP Groups

A SNMP Group is an entity that binds together SNMP Views and Security Model. A SNMP View is a subset of the MIB Tree. A Security Model is the set of security features that are used in order to accept a SNMP request.

### SNMP Views

A SNMP View is a subset of the entire MIB tree.\
By default, a Cisco Router already has a SNMP view defined, called _v1default_. This is the default view used for SNMPv1 and SNMPv2c when no custom views are configured.\
The _v1default_ view gives access to almost all MIB tree (iso = .1), except a fiew branches:

```
R# show snmp view
v1default iso - included permanent active
v1default internet.6.3.15 - excluded permanent active
v1default internet.6.3.16 - excluded permanent active
v1default internet.6.3.18 - excluded permanent active
v1default ciscoMgmt.394 - excluded permanent active
v1default ciscoMgmt.395 - excluded permanent active
v1default ciscoMgmt.399 - excluded permanent active
v1default ciscoMgmt.400 - excluded permanent active
```

You can define additional custom views with the command:

```
R(config)# snmp-server view VIEW MIB-BRANCH {included|excluded}
! included = includes the branch in the VIEW
! excluded = excludes the branch from the VIEW
```

### Security Models

SNMPv1 and SNMPv2 both use a Community based security model. This Security Model uses a clear text password (community string).\
SNMPv3 adds additional security features like integrity checks, authentication and encryption. There are 3 different Security Models for SNMPv3

| Model | Level        | Authentication                  | Encryption     |
| ----- | ------------ | ------------------------------- | -------------- |
| v1    | noAuthNoPriv | Community String                | No             |
| v2c   | noAuthNoPriv | Community String                | No             |
| v3    | noAuthNoPriv | Username, no Password           | No             |
| v3    | authNoPriv   | Username and Password (MD5/SHA) | No             |
| v3    | authPriv     | Username and Password (MD5/SHA) | DES, 3DES, AES |

#### **SNMP Communities (v1, v2c)**

The SNMP Agent for v1 and v2c must be configured with a community. SNMP Requests that are received by the agent must match one of the defined communities. When a community is defined, a SNMP Groups is automatically created for each community-based security model (v1 and v2c). This group binds the COMMUNITY to a ReadView (default: v1default), a WriteView (default: none) and a Security Model (v1/v2c).

```
R(config)# snmp-server community COMMUNITY [view VIEW] [ro|rw] [ACL]
! If no view is used, the default v1default view is used for bot v1 and v2c
! ro: assigns VIEW or v1default to ReadView and none to WriteView
! rw: assigns VIEW or v1default to both ReadView and WriteView
! ACL: SNMP messages denied by ACL are dropped
```

E.g:

```
R(config)# snmp-server community PUBLIC ro
R# show snmp group
groupname: PUBLIC                           security model:v1
readview : v1default                        writeview: <no writeview specified>
notifyview: <no notifyview specified>
row status: active

groupname: PUBLIC                           security model:v2c
readview : v1default                        writeview: <no writeview specified>
notifyview: <no notifyview specified>
row status: active
```

if you want to disable one of the versions you must disable the group for that version using:

```
R(config)# no snmp-server group GROUP {v1|v2c}
```

#### **User-Based Security Model (v3)**

The concept of SNMP Communities doesn’t apply to SNMPv3. For SNMPv3 you must configure the SNMP Group and SNMP users that are part of that group.

```
R(config)# snmp-server group GROUP-NAME v3 {noauth|auth|priv} [[read|write|notify] VIEW][access ACL]
! Default: ReadView = v1default, WriteView = none, NotifyView = none
```

* **noauth** – NoAuthNoPriv = No Authentication, No Privacy. No Authentication means only the username is checked. When defining a user for this group, do not set a password. No Privacy means that no encryption is used in the packets that are sent/received for this group.
* **auth** – AuthNoPriv = Authentication, No Privacy. Authentication means that the username and a password (stored as MD5 or SHA) are checked. When defining a user for this group, configure a password. No Privacy means that no encryption is used in the packets that are sent/received for this group.
* **priv** – AuthPriv = Authentication and Privacy. Authentication means that the username and a password (stored as MD5 or SHA) are checked. Privacy means that encryption (DES, 3DES or AES) is used in the packets that are sent/received for this group. When defining a user for this group, configure a password and an encryption method.

To configure a user, use:

```
! For NoAuth:
R(config)# snmp user USER NO-AUTH-GROUP v3 [access ACL]
! For Auth:
R(config)# snmp user USER AUTH-GROUP v3 auth {md5|sha} PASS [access ACL]
! For Priv
R(config)# snmp user USER PRIV-GROUP v3 auth {md5|sha} PASS priv {des|3des|aes {128|192|256}} [access ACL]
```

When specifying an Authentication password, you can use an already encrypted string:

```
R(config)# snmp user USER GROUP v3 encrypted auth {md5|sha} ENCRYPTED-PASS ...
! ENCRYPTED-PASS must be set as 32 hex characters in the format: HH:HH:...:HH
```

### Customizing SNMP Agent

You can set system information for the device, using the command:

```
R(config)# snmp-server {contact|location|chassis-id} STRING
```

These values will be accessible via the system mib of MIB2 and will help identify the device and who manages it.\
To enable the SNMP agent to accept and perform a shutdown (reload) request from the management station, use:

```
R(config)# snmp-server system-shutdown
```

To configure the maximum allowed SNMP packets, use:

```
R(config)# snmp-server packetsize MAX-BYTES
```

A router’s configuration can be saved to a TFTP file using a SNMP command. To limit the list of accepted TFTP files, use:

```
R(config)# snmp-server tftp-server-list ACL
```

To disable the SNMP Agent, use:

```
R(config)# no snmp-agent
```

When the agent is re-enabled, all previous config will be available.\
Upon each restart, a SNMP agent will assign a ID to each interface. This ID will be used as an index in different TableViews of the MIB, so it becomes important to have the same ID assigned to the same interface at each restart. You can enable ifIndex persistance globally or per interface, with:

```
! Globally:
R(config)# snmp-server ifindex persist
! Per interface:
R(config-if)# snmp ifindex persist
```

You can view the assigned values using:

```
R#sh snmp mib ifmib ifindex
```

## SNMP Traps and Informs

Traps and Informs are SNMP messages that are sent unsolicited by a SNMP Agent. The SNMP Manager listens for Traps or Informs on UDP port 162. Informs are similar to Traps but the SNMP Agent will resend them until they are acknowledged by the SNMP Manager. Traps are sent only once and no acknowledgement is expected.

You can set the interface used as the source of Traps/Informs with:

```
R(config)# snmp-server trap-interface INTERFACE
```

The events that can send notifications must be also globally enabled, using:

```
R(config)# snmp-server enable traps [NOTIFICATION-TYPE [NOTIFICATION-OPTIONS]]
! If NOTIFICATION-TYPE is not used then all traps are enabled
```

The agent must then be configured with the address and the of the management station

### SNMP v1 or v2c

For SNMPv1 or v2c:

```
R(config)# snmp-server host IP-ADDR [traps|informs] [version {1|2c}][COMMUNITY] [udp-port PORT] [NOTIFICATION-TYPE...]
! Default - SNMP Agent sends traps
! Default - SNMP version = 1
```

### SNMP v3

For version 3, use:

```
R(config)# snmp-server host IP-ADDR [traps|informs] version 3 {noauth|auth|priv} USER  [udp-port PORT][NOTIFICATION-TYPE...]
! Default - SNMP Agent sends traps
```

After you run this command, the router will automatically create a group with the same name as the USER. Then, you should define the snmp username as part of this group. (This is automatically done for noauth model).

```
R(config)# snmp-server username USER USER {noauth|auth|priv} ...
! The second USER is the GROUP Name
```

RMON can be configured to monitor a SNMP MIB Object and generate events when the value changes. You will have to define a RMON Alarm to define a monitored object:

```
R(config)# rmon alarm ALARM-ID MIB-OBJ INTERVAL {absolute|delta} rising-threshold RISING-VAL [RISING-EVENT] falling-threshold FALLING-VAL [FALLING-EVENT] [owner STRING]
! ALARM-ID = A number from 1 to 65535
! MIB-OBJ = The object that is monitored
! INTERVAL = Sampling Interval
! absolute|delta = Monitors the variation in the INTEVAL or the absolute value
! RISING-VAL = Value used as a rising threshold
! RISING-EVENT = Event generated by the rising threshold
! FALLING-VAL = Value used as falling threshold
! FALLING-EVENT = Event generated by the falling threshold
! owner = SNMP Alarm OWNER
```

The alarm can generate events based on the changes in the values of the monitored object. To define an event, use:

```
R(config)# rmon event EVENT [log] [trap COMMUNITY][description STRING][owner STRING]
! log = generates a log entry
! trap COMMUNITY = generates a trap with this SNMP COMMUNITY
```

To monitor RMON, use:

```
R# show rmon ...
```
