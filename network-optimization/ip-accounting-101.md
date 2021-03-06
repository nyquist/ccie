# IP Accounting 101

## Enabling IP Accounting

IP Accounting provides statistcs regarding the source and destination of the packets that are switched on an interface. Only outgoing traffic is accounted for and only traffic that is in transit, not generated by the router or that has the router as a destination.\
IP Accounting is enabled per interface

```
R(config-if)# ip accounting
! This command defaults to:
R(config-if)# ip accounting output-packets
```

You can configure an interface to account for the packets that are dropped by the access-group ACL on the interface, use:

```
R(config-if)# ip accounting access-violations
! Works for both incoming and outgoing packets
```

Also, you can configure an interface to account for the MAC Addresses of the frames received or sent on an interface:

```
R(config-if)# ip accounting mac-address {input|output}
```

Another option is to account for the IP Precedence value of the packets that pass through the interface:

```
R(config-if)# ip accounting precedence {input|output}
```

## System wide options

You can configure some system wide options for ip accounting, using:

```
R(config)# ip accounting-list HOST WILDCARD
! Limits the hosts for which accounting is enabled
R(config)# ip accounting-threshold MAX-ENTRIES
!Limits the max number of accounting entries
R(config)# ip accounting-transits COUNT
!Limits the number of transit entries
! Transit entires are those not matched by the accounting-list
```

## Monitoring IP Accounting

To see the results of IP accounting, use:

```
R#show ip accounting [checkpoint] [output-packets|access-violations]
   Source           Destination              Packets               Bytes
 3.3.3.3          12.0.0.1                       915               91500
 23.0.0.3         1.1.1.1                         80                8000
```

A checkpoint is a snapshot of the database before the last **clear ip accounting** was used.\
To see the mac address accounting, use:

```
R#show interfaces INTERFACE mac
FastEthernet0/1
  Input  (511 free)
    c200.1ddc.0001(2  ):  3787 packets, 425438 bytes, last: 896ms ago
                  Total:  3787 packets, 425438 bytes
  Output  (510 free)
    c200.1ddc.0001(2  ):  3547 packets, 404358 bytes, last: 572704ms ago
    0100.5e00.000a(85 ):  154 packets, 11396 bytes, last: 3348ms ago
                  Total:  3701 packets, 415754 bytes
```

To see the IP Precedence accounting, use:

```
R# show interface INTERFACE precedence
FastEthernet0/1 
Input
Precedence 0:  4 packets, 456 bytes
Output
Precedence 0:  4 packets, 456 bytes
```
