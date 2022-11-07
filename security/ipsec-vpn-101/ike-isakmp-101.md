# IKE / ISAKMP 101

## ISAKMP

The ISAKMP framework is a collection of methods used to manage the establishment of SAs and the keys involved in the process. IKE is a protocol that implements the ISAKMP framework.\
ISAKMP requires and IKE implements a 2 phase process for establishing an IPSec SA. In the 1st phase, an ISAKMP SA is established. The ISAKMP SA is then used to protect the negotiations for the IPSec SA in the 2nd phase.

There are 2 version of IKE: [IKEv1](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/217432-understand-ipsec-ikev1-protocol.html) (RFC2409) and [IKEv2](https://www.cisco.com/c/en/us/support/docs/security-vpn/ipsec-negotiation-ike-protocols/115936-understanding-ikev2-packet-exch-debug.html)(RFC 4306).&#x20;

The ISAKMP policy includes:

* An authentication method to ensure the identity of the peers
* An encryption method to protect the data and ensure privacy
* A HMAC (Hased Message Authentication Code) method to ensure the identiy of the sender and to ensure that the message has not been modified in transit
* A DH group (Diffie-Hellman) to determine the strenght of the encryption-key-algorithm. This algorithm is then used to derive the encryption and hash keys
* A session lifetime of the encryption key before it needs to be replaced.

ISAKMP separates the negotiation in 2 phases:

* Phase 1: The two peers establish a secure and authenticated tunnel. This tunnel is known as ISAKMAP SA. In this phase the tunnel protects the Control Plan between the peers. This communication runs on UDP 500 and UDP 4500. For IKEv2 there are 4 messages exchanged. For IKE v1 Phase 1 can run in two modes:
  * **IKE Main Mode**: Uses a minimum of 6 messages before the SA is established. Also provides identity protection.
  * **IKE Aggressive Mode**: Uses a minimum of 3 messages before the SA is established. Does not provide identity protection.
* Phase 2: In this pahse the peers negotiate key materials and algorithms for the encryption (SAs) of the data to be transfered over the IPSec tunnel. In this phase the tunnel protects the Data Plane and ESP is used to encapsulate and encrypt the traffic. This phase is also called Quick Mode
  * **IKE Quick Mode**: Similar to IKE Aggressive Mode, but protected by the ISAKMP SA negotiated in Phase 1

## Configuring ISAKMP Phase 1

By default, a Cisco router uses Main Mode in Phase 1, except when pre-shared keys are used. This can be disabled with:

```
R(config)# crypto isakmp aggressive-mode disable
```

In ISAKMP phase 1, the router looks up through the set of ISAKMP policies, in order of the SEQ.\
To define the ISAKMP policy, use the following commands:

```
R(config)# crypto isakmp policy SEQ
R(config-isakmp)# authentication {pre-share|rsa-sig|rsa-encr}
! Default: rsa-sig
R(config-isakmp)# encryption {des| 3des | aes {128|192|256}}
R(config-isakmp)# group {1|2|5}
! Sets the Diffie-Hellman group. Default: 1
R(config-isakmp)# hash {md5|sha}
! Default: SHA
R(config-isakmp)# lifetime SEC
! Default: 86400 (1 day)
```

There is a default is a isakmp policy that doesn't support pre-shared keys so when you want to use pre-shared keys you have to define a manual entry in the ISAKMP set of rules. You can see it with:

```
R# show crypto isakmp policy
```

### IKE Authentication

IKE authentication can be performed using pre-shared keys, PKI with digital certificates (X.509) or using RSA encrypted nonce. Depending on the authentication method you specified in the ISAKMP policy you will have to define one set of parameters or another.

#### **Pre-Shared Key**

Pre-shared keys are a simple way of authentication. To configure a pre-shared key, use:

```
R(config)# crypto isakmp key PSK {address PEER-IP [PEER-SUBNET]|hostname PEER-NAME} [no-xauth]
! no-xauth = Disbales extended authentication for router-to-router communication
```

#### **RSA encrypted nonces**

Follow these steps to generate RSA keys:\
First, set the hostname and domain:

```
R(config)# hostname HOSTNAME
R(config)# ip domain-name DOMAIN
```

```
R(config)# crypto key generate rsa usage-keys [modulus SIZE]
```

Verify the key exists with:

```
R# show crypto key mypubkey rsa
```

Now, you need to configure the public key-chain, with the public keys of the other peers:

```
R(config)# crypto key pubkey-chain rsa
Select the key based on either the name or the peer address:
R(config-pubkey-chain)# {addressed-key|named-key} PEER-IP [encryption|signature]
R(config-pubkey-key)# address ADDRESS
R(config-pubkey-key)# key-string
R(config-pubkey)# ...
! Enter the key of the PEER in HEX
R(config-pubkey)# quit
```

#### **PKI with digital certificates (X.509)**

This will be discussed in a later topic

## Additional ISAKMP configuration

To completely disable IKE, use:

```
R(config)# no crypto isakmp enable
```

You will have to manually specify all IPSEC SA information in the crypto maps.\
By default, each peer is identified by the IP address of the interface where the crypto map is applied. You can change this to either the hostname (used when the interface IP is not knwon – DHCP, or when multiple interfaces are used for IKE.) or to the DN (distinguished name – used for certificate-based authentication)

```
R(config)# crypto isakmp identity {address|hostname|dn}
! Default: address
```

To enable the recovery from “Invalid SPI” erros, use:

```
R(config)# crypto isakmp invalid-spi-recovery
! Default: disabled
```

To enable Dead Peer Detection, use:

```
R(config)# crypto isakmp keepalive SEC [RETRY-SEC] [periodic|on-demand]
! periodic - messages are sent every SEC. If 1 message is missed, the router sends messages every RETRY-SEC.
!     When 5 aggressive messages are missed, the peer is considered down.
! on-demand (default) - if no traffic is received from peer for SEC, then the retry process is started.
```

If IPSec is used with NAT, you can enable the router to send periodic keepalives just to keep the NAT translation active:

```
R(config)# crypto isakmp nat keepalive SEC
```
