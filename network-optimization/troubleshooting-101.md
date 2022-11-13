# Troubleshooting 101

## Troubleshooting CPU

<pre><code>R# show process cpu [sorted COLUMN]
<strong> CPU utilization for five seconds: 6%/3%; one minute: 7%; five minutes: 7%  
</strong> PID Runtime(ms)    Invoked   uSecs   5Sec   1Min   5Min TTY Process   
 115    16866996   43153481     490  0.72%  0.52%  0.52%   0 Skinny Msg Serve   
 163       88084 1683463169       0  0.38%  0.43%  0.41%   0 HQF Shaper Backg   
 129    26750192   94979172     281  0.38%  0.37%  0.33%   0 IP Input        
 ...</code></pre>

In this example the 5 seconds CPU utilziation is 6%/3% with the first percentage representing total CPU utilization and the second percentage representing interrupt utilization. Interrupt utilization is time spent by the CPU dealing with network packets (data plane). The difference between the 2 numbers is the ammount of time spent by the CPU on dealing with the other processes of the device. An interrupt percentage between 5-10% is acceptable. CPU spikes aren't neceserily bad but constantly keeping a high CPU utilization is a cause of concern.

## Troubleshooting memory

```
R# show memory
              Head    Total(b)     Used(b)     Free(b)   Lowest(b)  Largest(b) 
Processor 47C407A0   920385632   107682552   812703080   802714184   799891248
      I/O 3EA00000    23068672    22344664      724008      602976       82844
          Processor memory

  Address      Bytes     Prev     Next Ref  PrevF  NextF Alloc PC what 
47C407A0 0000000932 00000000 47C40B74 001 ------ ------ 40390618 *Packet Header* 
47C40B74 0000000932 47C407A0 47C40F48 001 ------ ------ 40390618 *Packet Header* 
47C40F48 0000000932 47C40B74 47C4131C 001 ------ ------ 40390618 *Packet Header* 
```

I/O memory is the memory used for temporary packet buffering. A device with low free memory can become slow or even reboot.

## Troubleshooting interfaces

```
R# show interface [INTF-ID]
R# show controllers [INTF-IF]
```

When troubleshooting interfaces, look for:

* MTU
* input queue drops - signify that the traffic is dropping because the router is receiving more traffic than it can handle
* Output queue drops - these are usuaully a result of congested links
* Input errors - these may be result of interface problems, duplex errors, CRC errors
* Output errors - these are usually the result of duplex issues

`show controllers` command providdes interface hardware statistics and may show different info based on controller type.

## Other troubleshooting commands

```
R# show platform
! Can be helpful when troubleshooting a router reload
```

