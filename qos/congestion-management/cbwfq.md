# CBWFQ

Class Based WFQ is configured using MQC. With CBWFQ, you use the same idea of [Weighted Fair Queueing](https://nyquist.eu/legacy-congestion-management/#22\_Weighted\_Fair\_Queuing), only this time, instead of using the IP\_Priority to assign a weight for each flow, like in [Flow-Based WFQ](https://nyquist.eu/legacy-congestion-management/#221\_Flow-Based\_WFQ), you can classify traffic on different criteria and then assign a weight for each class. Class weight is derived from the bandwidth that is configured in the policy.

## Classification

See [Classification using MQC](../classification-and-marking.md).

## Defining a CBWFQ Policy

```
R(config)# policy-map CBWFQ-POLICY
! Up to 64 classes can be used in a CBWFQ Policy:
R(config-pmap)# class CLASS
R(config-pmap-c)# bandwidth {KBPS | [remaining] percent PERCENT} 
! In a single policy you can use only values in kbps or only values in percent, not both
```

The bandwidth can be expressed as an absolute value in kbps, or as a percentage of the interface bandwidth or of the available remaining bandwidth. By default, only 75% of the interface bandwidth can be assigned. The remaining 25% is used by the router for L2 overhead, routing traffic and best-effort traffic (class-default). This limit can be changed per interface, using:

```
R(config-if)# max-reserved-bandwidth PERCENT
```

The value used for bandwidth can be found using:

```
R# show interface INTERFACE | i BW
  MTU 1500 bytes, BW 10000 Kbit, DLY 1000 usec,
```

And can be changed with the interface command:

```
R(config-if)# bandwidth KBPS
```

Remember there is a “default-class” that matches all the previously unmatched traffic. The bandwidth for this traffic is taken from the remaining 25% of reserved bandwidth, or whatever is left after the max-reserved-bandwidth is set.

The “default-class” uses FIFO queuing. However, this can be changed to Fair Queuing with the command:

```
R(config-pmap-c)# fair-queue
```

You can also configure the queue length for each class, using:

```
R(config-pmap-c)# queue-limit LENGTH
```

Once the queue-limit was reached, packets are dropped. You can use RED to prevent tail-drop.

### Low Latency Queuing (LLQ)

LLQ brings the feature of a strict priority queue to CBWFQ. The strict priority queue will always be serviced first. You can have more than one classes defined as priority classes, but they will all be assigned to the same priority queue. The other classes will be serviced in a normal CBWFQ fashion.\
In the event of congestion, policing is used, so that the priority queue will not use more than the bandwidth that was allocated.\
To define the priority queue, use:

```
R(config-pmap-c)# priority {KBPS | percent PERCENT} [BURST]
! The value used for bandwidth must include the L2 overhead
```

In a class that was configured with the **priority** command, you can’t use the **bandwidth** or **queue-limit** commands. Furthermore, when using LLQ inside a policy, the other classes can only set bandwidth as relative remaining percent. The default setting for Burst size is used to handle voice-like non-bursty traffic, but this can be changed, by setting BURST size.

LLQ is not compatible with WRED, but can work together with IP RTP Priority, which takes precedence.

## Enabling CBWFQ on an interface

You enable CBWFQ on an interface by specifying a service-policy where at least one class uses the bandwidth or the priority command. Not every command is available in a policy that sets CBWFQ as the queueing mechanism. Also, these policies can only be applied in the outgoing direction.\
To configure CBWFQ, use:

```
R(config-if)# service-policy output CBWFQ-POLICY
```

You can only enable CBWFQ on interfaces with default Queuing. This means FIFO (high speed interfaces) or WFQ (serial interfaces with speeds lower than E1 – 2048kbps).\
To verify the configuration, use:

```
R#show interface INTERFACE | b Queue
  Queueing strategy: Class-based queueing
  Output queue: 0/1000/64/0 (size/max total/threshold/drops)
     Conversations  0/1/256 (active/max active/max total)
     Reserved Conversations 2/2 (allocated/max allocated)
     Available Bandwidth 7116 kilobits/sec
! Same result as:
R# show queue INTERFACE
! or
R# show queuing interface INTERFACE
```

The interfaces configured for CBWFQ will also appear in the output of:

```
R# show queueing fair
Current fair queue configuration:

Interface           Discard    Dynamic  Reserved  Link    Priority
threshold  queues   queues    queues  queues
FastEthernet0/0     64         256      256       8       1
Serial1/0           64         256      0         8       1
Serial1/1           64         256      0         8       1

```

A more useful command is probably:

```
R1#sh policy-map interface f0/0
 FastEthernet0/0

  Service-policy output: CBWFQ-PERCENT

    Class-map: HTTP (match-all)
      0 packets, 0 bytes
      5 minute offered rate 0 bps, drop rate 0 bps
      Match: access-group 101
      Queueing
        Output Queue: Conversation 265
        Bandwidth 20 (%)
        Bandwidth 2000 (kbps)Max Threshold 64 (packets)
        (pkts matched/bytes matched) 0/0
        (depth/total drops/no-buffer drops) 0/0/0

    Class-map: SSH (match-all)
      0 packets, 0 bytes
      5 minute offered rate 0 bps, drop rate 0 bps
      Match: access-group 102
      Queueing
        Output Queue: Conversation 266
        Bandwidth 30 (%)
        Bandwidth 3000 (kbps)Max Threshold 64 (packets)
        (pkts matched/bytes matched) 0/0
        (depth/total drops/no-buffer drops) 0/0/0

    Class-map: class-default (match-any)
      184 packets, 18839 bytes
      5 minute offered rate 0 bps, drop rate 0 bps
      Match: any
```
