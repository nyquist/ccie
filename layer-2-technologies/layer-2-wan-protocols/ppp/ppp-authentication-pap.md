# PPP Authentication - PAP

PAP is a simple but not very secure authentication protocol. It sends the username and password information in clear text.

## One-way authentication

On the router that asks for authentication:

```
R1(config)# username USER password PASS
R1(config)# interface serial0/0
R1(config-if)# encapsulation ppp
R1(config-if)# ip address IP-ADDR1 MASK1
R1(config-if)# ppp authentication pap
R1(config-if)# no shut
```

On the router that is authenticating:

```
R2(config)# interface serial0/0
R2(config-if)# encapsulation ppp
R2(config-if)# ip address IP-ADDR2 MASK2
R2(config-if)# ppp pap sent-username USER password PASS
R2(config-if)# no shut
```

This is clearly a one-way authentication. Only R2 authenticates to R1.

## Two-way authentication

If two-way authentication is required, a configuration like the following should be used:

```
! On R1:
R1(config)# username USER1 password PASS1
R1(config)# interface serial0/0
R1(config-if)# encapsulation ppp
R1(config-if)# ip address IP-ADDR1 MASK1
R1(config-if)# ppp authentication pap
R1(config-if)# ppp pap sent-username USER2 password PASS2
R1(config-if)# no shut
! On R2:
R1(config)# username USER2 password PASS2
R2(config)# interface serial0/0
R2(config-if)# encapsulation ppp
R2(config-if)# ip address IP-ADDR2 MASK2
R2(config-if)# ppp authentication pap
R2(config-if)# ppp pap sent-username USER1 password PASS1
R2(config-if)# no shut
```

## Other Settings

By default a router will always respond to an authentication request even when no username is configured to be sent, but authentication will fail. Use the following command to refuse authentication requests:

```
R(config-if)#ppp pap refuse
```

### Debugging

When debugging, the best commands to use are:

```
R# debug ppp negotiation
R# debug ppp authentication
```

See this Cisco article about [debugging PPP neogtiation output](https://www.cisco.com/en/US/tech/tk713/tk507/technologies\_tech\_note09186a00800ae945.shtml)
