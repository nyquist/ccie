# ACLs 102

## Time-based ACLs

Define the time range:

```
R(config)# time-rage TIME-RANGE
R(config-time-range)# periodic DAYS-OF-WEEK HH:MM to [DAYS-OF-WEEK] HH:MM
! adds a recurring time to the time-range
! DAYS-OF-WEEK: daily (M-S), weekdays(M-F), weekend(S,S), Monday, Tuesday, ...
R(config-time-range)# absolute [start TIME DATE][end TIME DATE]
! adds an absoulte time to the time-range
! Only one absolute time is permitted in one time-range
```

Add the time-range to the ACL:

```
R(config)# access-list ACL {permit|deny} ... time-range TIME-RANGE
R(config-std-nacl)# {permit|deny} ... time-range TIME-RANGE
R(config-ext-nacl)# {permit|deny} ... time-range TIME-RANGE
```

## Reflexive ACLs

A reflexive ACL is used to permit outgoing traffic that was originated on one side of the connection (inside) and allow the returning packets from the other side (outside), but to deny traffic that was originated from the other side (outside). You can only use an extended named ACL to implement Reflexive ACLs

```
!Define the ACL that will permit traffic on the Inside:
R(config)# ip access-list extended ACL-OUTGOING
R(config-ext-nacl)# permit PROTOCOL SRC-IP WILDCARD DST-IP WILDCARD reflect REFLECT-NAME [timeout seconds]
! Defines the ACL that will permit traffic on the Outside:
R(config)# ip access-list extended ACL-INCOMING
R(config-ext-nacl)# evaluate REFLECT-NAME
```

The default timeout for dynamic entries in a reflexive ACL si 5 minutes but this can be changed per ACE or globally:

```
R(config)# ip reflexive-list timeout SEC
```

Usually, the ACL that matches outgoing traffic is set on the inside interface, while the ACL that evaluates the reflexive entries is set either on the outside interface on the incoming direction, or on the inside interface on the outgoing direction

```
! Apply OUTGOING-ACL
R(config)# interface INSIDE
R(config-if)# ip access-group ACL-OUTGOING in
! Apply INCOMING-ACL
R(config)# interface OUTSIDE
R(config-if)# ip access-group ACL-INCOMING in
! OR
R(config)# interface INSIDE
R(config-if)# ip access-group ACL-INCOMING out
```

## Dynamic ACLs – Lock-and-Key

This feature allows an IOS router do dynamically add ACEs in an ACL, in order to allow traffic for specific users. Users that need to pass traffic that is normally blocked by an ACL, can use telnet to logon to the router which then will dynamically add entries to the ACL in order to let them pass the filter.\
First, configure the ACL and include a dyanmic template:

```
R(config-ext-acl)# {permit|deny}...
R(config-ext-acl)# dynamic DYNAMIC-NAME [timeout MIN] {permit|deny} ...
```

The ACL should be applied on an interface but the dynamic ACEs will not be used in the ACL until a users authenticates itself using telnet. We will set this using the autocommand setting, defined on a line or on a username.

```
! Define user autocommand
R(config)# username USER autocommand access-enable [host] [timeout MIN]
! Define line autocommand
R(config-line)# autocommand access-enable [host] [timeout MIN]
```

The access-enable command will add the dynamic entries in the ACL. When using the **host** keyword, the ACE will only allow traffic from the host that connected via telnet. Otherwise, it will allow traffic from anybody.

Usually, you enable dynamic extension of the timeout period. If you login again, the timeout period is extended for each new login, otherwise the dynamic entries will be deleted from the ACE after they expire. To enable this feature, use:

```
R(config)# access-list dynamic-extend
```

You can manually clear dynamic entries using:

```
R# clear access-template ACL [DYNAMIC-NAME]
```
