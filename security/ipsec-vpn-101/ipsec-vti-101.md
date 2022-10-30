# IPSEC VTI 101

The VTI interface is a a Virtual Tunnel Interface that is protected via IPSEC. This simplifies the configuration of IPSEC and makes the logical configuration easier since this interface will work as any other interface on the router and will participate in the routing protocols just like another tunnel interface.

The line protocol of this interface will depend on the IPSec SA status

## IKE Policy (Phase1)

First we have to define the IKE peering parameters

```
R(config)# crypto isakmp policy PRIORITY
R(config-isakmp)# authentication {pre-share | rsa-encr | rsa-sig}
! Specifies authentication mode. Use pre-share for pre-shared keys.
R(config-isakmp)# hash {sha|sha256|sha384|sha512|md5}
! Specifies HMAC algorithm. Avoid MD5
R(config-isakmp)# encr {aes [128|192|256] | des | 3des}
! specifies encryption algorythm. Avoide DES
R(config-isakmp)# group DF-GRP
! specifies the DH group. Avoid 1 and 2.
R(config-isakmp)# lifetime {SEC}
! configures a lifetime before a new key needs to be generated. 
! Default: 86400 seconds (24 hours)
R(config-isakmp)# crypto isamkp key KEY-STRING address {PEER-IPV4 | ipv6 PEER-IPV6}

```

You can have multiple prorities thus defining several IKE combinations that are acceptable to be used.

To view the set or crypto policies use:

```
R# show crypto isakmp policy
```

See also [configuring ISAKMP](./#configuring-isakmp-phase-1)

## IPSEC Policy (Phase 2)

The next step is to define the [IPSEC Transform-set](ipsec-crypto-maps-101.md#define-a-transform-set)&#x20;

```
R(config)# crypto ipsec transform-set TRANSFORM-SET ENCR-ALG AUTH-ALG
R(cfg-crypto-trans)# mode {tunnel|transport [require]}
! Default: tunnel
```

To see the transform-sets, you can use:

```
R# show crypto ipsec transform-set [TRASFORM-SET]

```

## IPSEC Profile

The IPSEC Profile will be associated to a TRANSFORM-SET

```
R(config)# crypto ipsec profile IPSEC-PROFILE
R(ipsec-profile)# set transform-set TRANSFORM-SET
```

## Create IPSec Tunnel

To create an IPSec tunnel interface you can use these commands:

```
R(config)# interface Tunnel0
R(config-if)# tunnel source {INTF-NAME| SRC-IP}
R(config-if)# tunnel destination DEST-IP
R(config-if)# tunnel mode ipsec {ipv4|ipv6}
R(config-if)# tunnel protection ipsec profile IPSEC-PROFILE
```

## Monitor&#x20;

To monitor IPSEC SA (Phase 2) you can use this command:

```
R# show crypto ipsec sa [interface INTF-NAME | peer PEER-IP]
! Look for pkts count to see if anything is sent using the SA
```

Then you should verify the status of the Tunnel interface since the line-protocol depends on the SA status and then see if there is any route that would send any traffic over this interface.

To monitor IKE SA (Phase 1) you can use this command:

```
R# show crypto ike sa [detail]

```

See here for more info: [https://www.cisco.com/c/en/us/tech/security-vpn/ipsec-negotiation-ike-protocols/tsd-technology-support-troubleshooting-technotes-list.html](https://www.cisco.com/c/en/us/tech/security-vpn/ipsec-negotiation-ike-protocols/tsd-technology-support-troubleshooting-technotes-list.html)
