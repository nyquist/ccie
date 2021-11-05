# Backup Interfaces

## In Theory

A backup interface is an interface that stays inactive as long as the primary interface is in “up/up” state, but becomes active when the primary interface’s protocol status becomes “down”.\
When the protocol on the primary interface comes back up, the backup interface moves back in a “standby mode”.\
You should know that if the primary interface is administratively down, the backup interface won’t come up. So to test, you have to do something on the other end.\
The backup interface can be configured to come up not only when the primary interface is down, but also when it’s utilization reaches a certain threshold:

```
R(config-if)# backup load {enable_threshold|never} {disable_load|never}
```

To prevent link flapping you can delay the switchover using:

```
R(config-if)# backup delay {enable_delay|never} {disable_delay|never}
```

## In Practice

Here’s an example:

[![Backup Interface Example](https://nyquist.eu/wp-content/uploads/2012/02/BackupInterface.png)](https://nyquist.eu/wp-content/uploads/2012/02/BackupInterface.png)

Backup Interface Example

R1 and R2 are connected over Fa0/0 and Seria1/0 interfaces. We will configure Fa0/0 with ip addresses in 12.0.0.0/24 range S1/0 with addresses in 21.0.0.0/24 range. To test connectivity we will enable one loopback on each router and start rip to to advertise the routes from one to another

```
!On R1:
R1(config)# interface Fa0/0
R1(config-if)# ip address 12.0.0.1 255.255.255.0
R1(config-if)# no shut
R1(config-if)# exit
R1(config)# interface Serial1/0
R1(config-if)# ip address 21.0.0.1 255.255.255.0
! On newer IOS version, no need to specify clock rate on the DCE
R1(config-if)# no shut
R1(config)# interface Lo0
R1(config-if)# ip address 1.1.1.1 255.255.255.255
R1(config-if)# exit
R1(config-if)# router rip
R1(config-router)# network 0.0.0.0
!On R2:
R2(config)# interface Fa0/0
R2(config-if)# ip address 12.0.0.2 255.255.255.0
R2(config-if)# no shut
R2(config-if)# exit
R2(config)# interface Serial1/0
R2(config-if)# ip address 21.0.0.2 255.255.255.0
R2(config-if)# no shut
R2(config)# interface Lo0
R2(config-if)# ip address 2.2.2.2 255.255.255.255
R2(config-if)# exit
R2(config-if)# router rip
R2(config-router)# network 0.0.0.0
```

Shortly, we should be able to ping each router’s loopback interface, from the other one:

```
!On R1:
R1#ping 2.2.2.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 12/18/24 ms
!On R2:
R2#ping 1.1.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 16/20/24 ms
```

Let’s verify the routing tables, also.

```
On R1:
R1#sh ip route
1.0.0.0/32 is subnetted, 1 subnets
C 1.1.1.1 is directly connected, Loopback0
R 2.0.0.0/8 [120/1] via 21.0.0.2, 00:00:15, Serial1/0
[120/1] via 12.0.0.2, 00:00:04, FastEthernet0/0
21.0.0.0/24 is subnetted, 1 subnets
C 21.0.0.0 is directly connected, Serial1/0
12.0.0.0/24 is subnetted, 1 subnets
C 12.0.0.0 is directly connected, FastEthernet0/0
!On R2:
R2#sh ip route
R 1.0.0.0/8 [120/1] via 21.0.0.1, 00:00:15, Serial1/0
[120/1] via 12.0.0.1, 00:00:06, FastEthernet0/0
2.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C 2.2.2.2/32 is directly connected, Loopback0
R 2.0.0.0/8 [120/1] via 12.0.0.1, 00:02:27, FastEthernet0/0
21.0.0.0/24 is subnetted, 1 subnets
C 21.0.0.0 is directly connected, Serial1/0
12.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C 12.0.0.0/24 is directly connected, FastEthernet0/0
R 12.0.0.0/8 [120/1] via 21.0.0.1, 00:03:07, Serial1/0
```

Notice the 2 routes installed for the loopback addresses, one for each physical link.

### The good interface

Now let’s enable Fa0/0 as backup for S1/0 on R1. Let’s also start debugging on R1:

```
R1# debug backup
R1# conf t
R1(config)# interface S1/0
R1(config-if)# backup interface Fa0/0
```

As soon as we set Fa0/0 as the backup interface of S1/0, the backup interface goes down, in standby mode:

```
*Mar  1 01:00:25.015: BACKUP(Serial1/0): changed state to "initializing"
*Mar  1 01:00:25.015: BACKUP(Serial1/0): secondary interface (FastEthernet0/0) configured
*Mar  1 01:00:27.015: BACKUP(Serial1/0): event = timer expired on primary
*Mar  1 01:00:27.019: BACKUP(Serial1/0): secondary interface (FastEthernet0/0) moved to standby
*Mar  1 01:00:27.023: BACKUP(Serial1/0): changed state to "normal operation"
*Mar  1 01:00:29.019: %LINK-5-CHANGED: Interface FastEthernet0/0, changed state to standby mode
*Mar  1 01:00:30.019: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/0, changed state to down
*Mar  1 01:00:30.019: BACKUP(FastEthernet0/0): event = secondary interface went 
```

We can see the status with:

```
R1# show ip interface brief
Interface IP-Address OK? Method Status Protocol
FastEthernet0/0 12.0.0.1 YES manual standby mode down
Serial1/0 21.0.0.1 YES manual up up
Loopback0 1.1.1.1 YES manual up up
```

Now let’s shut down the serial link on R2:

```
R2(config)# interface serial0/0
R2(config-if)# shut
```

Now, based on the keepalive mechanism, the serial link on R1 will move the link into an “up/down” state in about 30 seconds (3 missed keepalives) and will move the backup interface in forwarding mode:

```
*Mar  1 01:08:53.439: %LINEPROTO-5-UPDOWN: Line protocol on Interface Serial1/0, changed state to down
*Mar  1 01:08:53.443: BACKUP(Serial1/0): event = primary interface went down
*Mar  1 01:08:53.443: BACKUP(Serial1/0): changed state to "waiting to backup"
*Mar  1 01:08:53.447: BACKUP(Serial1/0): event = timer expired on primary
*Mar  1 01:08:53.459: BACKUP(Serial1/0): secondary interface (FastEthernet0/0) made active
*Mar  1 01:08:53.459: BACKUP(Serial1/0): changed state to "backup mode"
*Mar  1 01:08:55.447: %LINK-3-UPDOWN: Interface FastEthernet0/0, changed state to up
*Mar  1 01:08:56.447: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/0, changed state to up
*Mar  1 01:08:56.447: BACKUP(FastEthernet0/0): event = secondary interface came up
R1#sh ip int brie
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            12.0.0.1        YES manual up                    up
Serial1/0                  21.0.0.1        YES manual up                    down
Loopback0                  1.1.1.1         YES manual up                    up
```

The routing protocol converges and we can ping 2.2.2.2 from 1.1.1.1

```
R1#sh ip route
Gateway of last resort is not set

     1.0.0.0/32 is subnetted, 1 subnets
C       1.1.1.1 is directly connected, Loopback0
R    2.0.0.0/8 [120/1] via 12.0.0.2, 00:00:13, FastEthernet0/0
     12.0.0.0/24 is subnetted, 1 subnets
C       12.0.0.0 is directly connected, FastEthernet0/0
R1#ping 2.2.2.2 source 1.1.1.1

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2.2.2.2, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 16/18/24 ms
```

When we bring back up the serial interface on R2, Serial 1/0 will come up on R1 and Fa0/0 will move to standby mode again:

```
R2(config)#int s1/0
R2(config-if)#no shut
!On R1:
*Mar  1 01:18:43.423: %LINEPROTO-5-UPDOWN: Line protocol on Interface Serial1/0, changed state to up
*Mar  1 01:18:43.431: BACKUP(Serial1/0): event = primary interface came up
*Mar  1 01:18:43.431: BACKUP(Serial1/0): changed state to "waiting to revert"
*Mar  1 01:18:43.439: BACKUP(Serial1/0): event = timer expired on primary
*Mar  1 01:18:43.443: BACKUP(Serial1/0): secondary interface (FastEthernet0/0) moved to standby
*Mar  1 01:18:43.443: BACKUP(Serial1/0): changed state to "normal operation"
*Mar  1 01:18:45.443: %LINK-5-CHANGED: Interface FastEthernet0/0, changed state to standby mode
*Mar  1 01:18:46.443: %LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/0, changed state to down
*Mar  1 01:18:46.443: BACKUP(FastEthernet0/0): event = secondary interface went
R1#sh ip int brie
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            12.0.0.1        YES manual standby mode          down
Serial1/0                  21.0.0.1        YES manual up                    up
Loopback0                  1.1.1.1         YES manual up                    up 
```

### The bad interface

Things worked as expected when we set a backup for the serial interface. Now let’s try setting the serial interface as the backup for the ethernet interface:

```
R1(config)# interface serial1/0
R1(config-if)# no backup interface
R1(config-if)# exit
R1(config)# interface Fa0/0
R1(config-if)# backup interface serial1/0
R1(config-if)# end
R1# show ip int brie
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            12.0.0.1        YES manual up                    up
Serial1/0                  21.0.0.1        YES manual standby mode          down
Loopback0                  1.1.1.1         YES manual up                    up  
```

Thinks work as expected, now let’s shut the FastEthernet interface on R2:

```
R2(config)# interface fa0/0
R2(config-if)# shut
```

And now we wait…\
When you have waited long enough, you shoud have noticed that the FastEthernet interface on Fa0/0 never went down. The keepalive mechanism on Ethernet links is not used to test connectivity with another host, but to see if the interface can send and receive Ethernet frames. This is because Ethernet links are not considered point-to-point interfaces and they are expected to find more than one neighbor on the link. Since the link will always be up, the backup interface will remain in standby mode and will not be used for forwarding.

The same thing would happen with other Multipoint interfaces, like the Frame Relay physical interface or the multipoing subinterface. A point-to-point subinterface would move the protocol status to down when the DLCI assigned to it si not active.

The solution here is to use a more advanced tracking system, like **Enhanced Object Tracking**
