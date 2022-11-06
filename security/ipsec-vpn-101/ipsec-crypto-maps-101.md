# IPSEC Crypto Maps 101

## Define a transform-set

A transform-set contains a list of possible transformations that are used in the IKE negotiation for the IPSec SA (IKE phase 2).

```
R(config)# crypto ipsec transform-set TRANSFORM-SET [AH-TR] [ESP-ENC-TR] [ESP-AUTH-TR] [COMPRESS-TR]
! AH-TR: ah-md5-hmac, ah-sha-hmac
! ESP-ENC-TR: esp-3des, esp-aes, esp-des, , esp-null, esp-seal,
! ESP-AUTH-TR: esp-sha-hmac, esp-md5-hmac
! COMPRESS-TR: comp-lzs
```

Once you define the transform-set, you can also specify the mode: tunnel mode or trasport mode

```
R(cfg-crypto-trans)# mode {tunnel|transport}
!Changes the mode associated with the transform set.
```

The mode setting is applicable only to the traffic whose source and destination addresses are the IPsec peer addresses. All other traffic is in tunnel mode only.

## Define Crypto Maps

### **Static Maps: IKE vs manual**

When you create a crypto map, you have 2 options: either use manual keying, or IKE. To create a crypto map, use:

```
R(config)# crypto map CRYPTO-MAP SEQ {ipsec-isakmp|ipsec-manual}
! ipsec-isakmp uses IKE, ipsec-manual uses manual keys
R(config-crypto-map)# match address ACL
! Use the ACL to match the interesting traffic
R(config-crypto-map)# set peer PEER-IP
R(config-crypto-map)# set transform-set TRANSFORM-SET
```

When using IKE, you can also configure these settings:

```
R(config-crypto-map)# set security-association lifetime {seconds SEC| kilobytes KB}
! Time/Quantity of data after which new keys are generated
R(config-crypto-map)# set security-association idle-time SEC
! When no packets are sent/received for idle-time, the SA is deleted.
R(config-crypto-map)# set security-association level per-host
! By default, all traffic for a peer uses the same SA.
! When using this command, packets for each host/destination pair will use a different SA
R(config-crypto-map)# set security-association replay {window-size SIZE|disable}
! You can customize or disable the anti-replay feature of IPSec.
R(config-crypto-map)# set pfs [group1|group2|group5]
! By default PFS is not requested. If requested, with no param, group1 is used.
```

When using manual keying, you have to specify the keys with:

```
! For ESP:
R(config-crypto-map)# set session-key {inbound|outbound} esp SPI cipher HEX-KEY [authenticator HEX-AUTHENTICATOR]
! For AH:
R(config-crypto-map)# set session-key {inbound|outbound} ah SPI HEX-KEY
```

Of course, the inbound key on one side must match the outbound key on the other side

### **Dynamic maps**

A third option is to use dynamic maps. Dynamic maps act as templates that do not have all the information configured, but rather the missing data will be negotiated with the other peer. This implies that dynamic maps require the use of IKE. A dynamic map will never be used to start a negotiation. It will only be used to reply to other peers’requests.\
First you need to define the dynamic crypto map template with:

```
R(config)#crypto dynamic-map DYNAMIC-CRYPTO-MAP SEQ
R(config-crypto-map)#...
```

Then, reference the template in a static crypto map with:

```
R(config)# crypto map CRYPTO-MAP 10 ipsec-isakmp dynamic DYNAMIC-CRYPTO-MAP [discover]
! discover - enables Tunnel Endpoint Discovery
```

### **Additional crypto map settings**

```
R(config-crypto-map)#qos pre-classify
! Enables qos classification before the packet is encrypted
R(config-crypto-map)#set ip access-group ACL
! Enables ACL filtering of the traffic before encryption. The interface access-group won't work on encrypted packets
```

RRI (Reverse Route Injection) is a feature that allows one peer to add static routes for the subnets on the other peer. By default, routes are added only if packets for the specified destination actually flow through the router.

```
R(config-crypto-map)# reverse-route {tag TAG|remote-peer [NEXT-HOP]} [static]
! no params: the route for the subnet will point to the interface where the crypto map is configured
! remote-peer - the route for the subnet will point to the peer address
!             - an additional route for the peer will point to the interface where the crypto map is configured
! remote-peer NEXT-HOP - the route for the subnet will point to the NEXT-HOP address
!                      - no additional route is added
! tag TAG - will add the routes with a TAG that can be used for filtering when redistributing
! static - creates the routes only based on ACL. There may be no active flows for that destination.
```

## Apply the Crypto Map

When sending traffic out an interface, the router will apply the IPSec configuration only if the outgoing interface has a crypto map configured, and that crypto map matches the outgoing packet to be sent. To configure an interface for IPSec, apply the crypto map with:

```
R(config-if)# crypto map CRYPTO-MAP
```

