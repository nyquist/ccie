# PPP Authentication – CHAP

## CHAP algorithm

CHAP is more secure than PAP because even though it sends usernames in clear text, it uses an MD5 hash instead of a clear text password for authentication.

When a router is set to ask for authentication, it will send a CHAP Challenge to the other router which contains an **ID**, a **random number** and a **username**. By default, the username used is the hostname of the router. For each Challenge, the router will store in memory the random number used.

When the router that shoud authenticate receives the Challenge, it must search in it’s database a password for the supplied username. If it doesn’t find one, the authentication will fail. If if finds one, the router will use the **ID**, **the random number** and the **password** to generate an **MD5 hash**. It will send this MD5 hash in a Response packet, together with a **username** to the router that issued the authentication challenge.The username is by default the hostname of the router that responds

Upon receiving the Response, the first router uses the ID to determine what Challenge the Response belongs to and retrives the random number used in the Challenge it sent. It then searches for a password coresponding to the username received from the other router. If no match is found, authentication fails. If a match is found, then the router uses the **ID**, **the random number** it used in the Challenge and the **password** it found to generate an **MD5 hash**. If the MD5 hash it generates matches the one that it received, then it (statisticly) means both routers used the same password and authentication succeeds.

## One-way authentication

The configuration for one-way authentication should look like this:

```
!On R1
R1(config)# interface serial0
R1(config-if)# ip address IP-ADDR1 MASK1
R1(config-if)# encapsulation ppp
R1(config-if)# ppp authentication chap
R1(config-if)# exit
R1(config)# username R2 password PASS
!On R2
R2(config)# interface serial0
R2(config-if)# ip address IP-ADDR2 MASK2
R2(config-if)# encapsualation ppp
R2(config-if)# no shut
R2(config-if)# exit
R2(config)# username R1 password PASS
```

Even though this might look like a two way authentication, it isn’t. For two-way authentication both routers should send a Challenge.

## Two-way authentication

```
!On R1
R1(config)# interface serial0
R1(config-if)# ip address IP-ADDR1 MASK1
R1(config-if)# encapsulation ppp
R1(config-if)# ppp authentication chap
R1(config-if)# exit
R1(config)# username R2 password PASS
!On R2
R2(config)# interface serial0
R2(config-if)# ip address IP-ADDR2 MASK2
R2(config-if)# encapsualation ppp
R2(config-if)# ppp authentication chap
R2(config-if)# no shut
R2(config-if)# exit
R2(config)# username R1 password PASS
```

As you can see, when using two way authentication you cannot use different passwords, as was possible with PAP.

## More settings

To use a different username than the configured hostname, you can use:

```
R(config)# username USER2 password PASS2
R(config)# interface serial0
R(config-if)# ppp chap hostname USER2
```

and to use a default password for unknown usernames, you can use:

```
R(config)# interface serial0
R(config-if)# ppp chap password PASS
```

By default a router will always respond to an authentication request with its hostname as a username, even if there is no password configured. Of course, authentication will fail. Use the following command to refuse authentication requests:

```
R(config-if)#ppp chap refuse
```

## Debugging

When debugging, the best commands to use are:

```
R# debug ppp negotiation
R# debug ppp authentication
```

See this Cisco article about [debugging PPP neogtiation output](https://www.cisco.com/en/US/tech/tk713/tk507/technologies\_tech\_note09186a00800ae945.shtml)
