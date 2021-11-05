# PPP Authentication – EAP

## One way authentication

### EAP Server (Authenticator)

```
! Set encapsulation to PPP
R(config-if)# encapsulation ppp
! Enable PPP authentication
R(config-if)# ppp authentication eap
```

PPP will require the use of a Radius. However, you can use the local usernames if you configure:

```
R(config-if)# ppp eap local
! Use local username and passwords to perform authenticaion
```

Remember to configure the username and password that the client will use:

```
R(config)# username USER password PASS
```

### EAP Client (Authenticating)

```
R(config-if)# encapsulation ppp
! Set the password used for EAP
R(config-if)# ppp eap password PASS
! By default, the hostname is used as the username. You can change it with:
R(config-if)# ppp eap identity USER
```

## Two way authentication

On R1:

```
R1(config)# username USER2 password PASS2
R1(config)# interface INTERFACE
R1(config-if)# encapsulation ppp
R1(config-if)# ppp authentication eap
R1(config-if)# ppp eap local
R1(config-if)# ppp eap password PASS1
R1(config-if)# ppp eap identity USER1
```

On R2:

```
R1(config)# username USER1 password PASS1
R2(config)# interface INTERFACE
R2(config-if)# encapsulation ppp
R2(config-if)# ppp authentication eap
R2(config-if)# ppp eap local
R2(config-if)# ppp eap password PASS2
R2(config-if)# ppp eap identity USER2
```
