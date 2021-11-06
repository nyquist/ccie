# Header and Payload Compression

## Header Compression (RTP,TCP)

Header compression works by creating a context on both sides of a connection for each TCP/RTP data stream. The context contains header information for the stream so that data doesn’t need to be sent with every packet. For RTP, you can also send full headers periodically to refresh information. Header compression is only available for TCP and RTP.

### Interface level Header Compression

#### **PPP, HDLC**

To enable header compression,use:

```
R(config-if)# ip {rtp|tcp} header compression [passive|ietf-format|iphc-format] [periodic-refresh]
! periodic-refresh - RTP only
```

When using the **passive** keyword, compression will be used only for destinations that send compressed headers. Otherwise, all outgoing traffic will be compressed.\
Periodic refreshes means sending full headers from time to time, but this is available only for RTP\
The total number of context can be configured with:

```
R(config)# ip {rtp|tcp} compression-connections CONTEXTS
```

Take into account that one context is used for each directions, so a bidirectional communication will use twice as much contexts if configured on both sides.\
Additional settings can be made with:

```
R(config-if)# ip header-compression ?
  disable-feedback  Switch off context status mechanism
  max-header        Maximim compressible header
  max-period        Number of Packets between refresh
  max-time          Time between refresh
  recoverable-loss  Maximum recoverable loss
```

To see the status of header compression, use:

```
R# show ip {rtp|tcp} header-compression
```

#### **Frame Relay**

On Frame Relay interfaces, TCP Header compression is only available on interfaces where the encapsulation used is cisco, not ietf. Header compression can be configured per interface or subinterface, using commands that start with **frame-relay ip {rtp|tcp} header-compression**.

```
R(config-if)# frame-relay ip {rtp|tcp} header-compression [passive][periodic-refresh]
R(config-fi)# frame-relay ip {rtp|tcp} compression-connections CONTEXTS
```

When you configure on a multilink (sub)interface, header compression applies to all DLCIs. You can specifically disable compression on some DLCIs, using:

```
R(config-if)# frame-relay map PROTOCOL ADDRESS DLCI nocompress
```

If you only want to enable on some DLCIs, use:

```
R(config-if)# frame-relay map PROTOCOL ADDRESS DLCI compress {active|passive} [connections CONTEXTS] [periodic-refresh]
```

### Class Based Header Compression

You can also configure header compression inside an outgoing policy for traffic matched by a class defintion. You should use the following command to enable header compression:

```
R(config-pmap-c)# compression [header [ip [rtp|tcp]]]
! If you do not specify rtp or tcp, both are compressed
! The policy-map can pe applied only outgoing
```

### IPHC Profile

Another option is to use IPHC Profiles. First define the profile:

```
R(config)#iphc-profile PROFILE-NAME {ietf|van-jacobsen}
```

Then set compression options:

```
R(config-iphc)# ?
IPHC Profile configuration commands:
  exit              Exit from IPHC Profile configuration mode
  feedback          enable feedback
  maximum           set the limit for IPHC options
  no                Negate or set default values of a command
  non-tcp           enable non-tcp header-compression
  recoverable-loss  ECRTP Recoverable loss
  refresh           context refresh options
  rtp               enable rtp header-compression
  tcp               enable tcp header-compression
```

In the end apply the profile per interface:

```
R(config-if)# iphc-profile PROFILE-NAME
```

To verifu, use:

```
R# sh iphc-profile [PROFILE-NAME]
```

Payload compression is available on serial interfaces and it depends on the type of encapsulation. Most payload compression techniques are based on the Stacker or the Predictor algorithm:\
Stacker algorithm tries to replace big chunks of data with index in a dictionary. It requires less memory and more CPU, it is more efficient, but less faster.\
Predictor algorithm tries to predict the next character. It requires more memory and less CPU, it is less efficient but is faster.\
When determining QoS byte counts, legacy QoS uses the data before compression, while the more modern MQC uses the data after compression.\
To monitor compression, use:

```
R# show compress [details]
```

Payload compression options differ based on the encapsulation used:

### HDLC

```
! HDLC:
R(config-if)# encapsulation hdlc
R(config-if)# compress ?
  stac  stac compression algorithm
```

### PPP

```
R(config-if)# encapsulation ppp
R(config-if)# compress ?
  lzs        lzs compression type
  mppc       MPPC compression type
  predictor  predictor compression type
  stac       stac compression algorithm
```

### Frame Relay

Per interface or subinterface:

```
R(config-if)# frame-relay payload-compression ?
  FRF9              FRF9 encapsulation - standard based
  data-stream       cisco proprietary encapsulation
  packet-by-packet  cisco proprietary encapsulation (similar to Predictor)
```

Per VC:

```
R(config-if)# frame-relay map PROTOCOL ADDRESS DLCI payload-compression ?
  FRF9              FRF9 encapsulation - standard based
  data-stream       cisco proprietary encapsulation
  packet-by-packet  cisco proprietary encapsulation (similar to Predictor)
```
