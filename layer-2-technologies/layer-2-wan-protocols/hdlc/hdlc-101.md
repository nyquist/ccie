# HDLC 101

## Basic Configuration

The default encapsulation on serial interfaces on a Cisco Router is HDLC.\
Even though HDLC exists as an open standard, Cisco routers use a modified version of the original standard that includes a new field in the header used for identification of the upper layer protocol that is encapsulated. Many other vendors can interoperate with Cisco Routers even when using this version of the protocol, but technically it is considered a non-standard implementation. If the requirements are to implement a standard encapsulation on a Serial interface, a better option would be PPP.

Since HDLC is the default encapsulation, configuration is probably the easiest ever:

```
! On R1:
R1(config)# interface Serial0/0
R1(config-if)# ip address 99.0.0.1 255.255.255.0
R1(config-if)# no shut
! On R2:
R1(config)# interface Serial0/0
R1(config-if)# ip address 99.0.0.2 255.255.255.0
R1(config-if)# no shut
! Test
R1# ping 99.0.0.2
!!!!!
```

Normally, when connecting two routers on a serial link, one of them must be the DTE, the other must be the DCE.\
The DCE router must set a clock rate for the interface using:

```
R1(config-if)# clock rate CLOCK-RATE
```

In newer versions of the IOS, this is not needed anymore because by default, every serial interface has a clock rate set, but only the DCE end will use it.

How do we know which is the DCE end? The easiest method is to look at the command:

```
R1# show controllers serial0/0 | i DCE|DTE
cable type : V.11 (X.21) DCE cable, received clockrate 2015232
```

## Keepalives

Cisco HDLC uses keepalives to monitor the link state. By default, keepalives are sent and expected every 10 seconds. Three missed keepalives will move the interface protocol to a “down” status.

```
R1(config-if)#keepalive ?
< 0-32767> Keepalive period (default 10 seconds)
< cr>
```

Hitting enter will set the default value of 10 seconds. To disable keepalives completly use:

```
R1(config-if)# no keepalive
```

To verify keepalive value use:

```
R1# show interface serial0/0 | i alive
Keepalive set (10 sec)
```

## Compression

HDLC can be configured to use software compression using the Stacker(LZS) algorithm. Compression must be enabled on both ends of the link:

```
!On R1
R1(config-if)# compress stack
!On R2
R2(config-if)# compress stack
```

Compression can be verified using:

```
R1# show compress [detail]
Serial0/0
Software compression enabled
uncompressed bytes xmt/rcv 101850/100816
compressed bytes xmt/rcv 43041/42357
Compressed bytes sent: 43041 bytes 3 Kbits/sec ratio: 2.366
Compressed bytes recv: 42357 bytes 3 Kbits/sec ratio: 2.380
1 min avg ratio xmt/rcv 2.152/2.139
5 min avg ratio xmt/rcv 2.150/2.137
10 min avg ratio xmt/rcv 2.150/2.137
no bufs xmt 0 no bufs rcv 0
resyncs 0
Additional Stac Stats:
Transmit bytes: Uncompressed = 220 Compressed = 43041
Received bytes: Compressed = 46381 Uncompressed = 0
```

Notice the compression ratio. One byte was sent after compression for each 2.366 bytes of uncompressed data. In the other direction, 1 byte of data was received for each uncompressed 2.380 bytes of date. These are of course average values. Beware that compression can affect system performance!
