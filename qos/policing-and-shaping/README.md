# Policing and Shaping

## Token Bucket

The token bucket is the analogy used to describe the rate of transfer. In order to send data, we have a bucket that regularly fills with tokens. When we want to send data, we must consume 1 token for every bit (or byte, depending on how we define the tokens) that we send. If we need to send data and we don’t have enough tokens in the bucket then we can drop the packet (Policing) or wait for a refill and try again (Shaping).

### Single Rate, Two Color Marker

A Token Bucket is described by the size of the bucket, and the rate at which the bucket fills with tokens. The size of the bucket is known as Bc (Committed Burst or Burst Size), usually measured in Bytes and the rate is called CIR (Committed Information Rate), measured in bps. Normally the flow of tokens would be continuous, but we are dealing with discreet values here, so we have to consider that over an interval equal to Tc, we will receive Tc\*CIR tokens. Those tokens must fill the bucket so here’s the formula that bounds them all together:\
Bc = Tc \* CIR\
In the Single Rate, Two Color Marker, we have a Single Rate – CIR, and two available color for the packets: green – traffic below Bc (conforming), or red – traffic above Bc (exceeding)

### Single Rate, Three Color Marker

If we would send enough bytes to consume all the tokens, all the time, the transmission rate would be equal to CIR. But most of the time, due to the bursty nature of traffic, this is not the case. Therefor, most of the time our rate will be below CIR.\
In order to have an average as close to CIR, we can send more than Bc in one timeslot, as long as we don’t exceed the average value of CIR. This is done using a second bucket, that collects excess tokens. When the first bucket is full of tokens and we receive new tokens, it means that we didn’t use all our available tokens to send data. In the “Single Rate, Two Color Marker”, these tokens would be lost. In the “Single Rate, Three Color Marker”, these tokens can be collected in the Excess Bucket of size Be (Excess Burst). In one Tc interval, we can send up to Bc+Be if the CIR is not exceeded. This implementation will allow our average rate to get closer to CIR, but still not exceeding it, since Be is only available if the first bucket is not entirely emptied. Cisco recommends using these values:\
\[pmath]B\_c = CIR \* 1.5s \* {{1 Byte}/{8 bits}}\[/pmath]\
\[pmath]B\_e = 2\*Bc\[/pmath]\
Now, we have three types of traffic: Traffic that is below Bc (conform – green), traffic that is between Bc and Bc+Be (exceed – yellow) and traffic above Be (violate – red)

### Two Rate, Three Color Marker

Another option is to use 2 rates, a CIR and a PIR (Peak Information Rate). You can send traffic as long as you never send more than the PIR, and the average rate stays below CIR. This is done also with two buckets, but each one is filled at its own pace. The first bucket, size Bc, fills at CIR rate, and the second one, size Be, fills at PIR rate, but only when the first one is full.\
We have three types of traffic again: Traffic that is below CIR (conform – green), traffic that is between CIR and PIR (exceed – yellow) and traffic above PIR (violate – red)

Policing is a method of limiting the traffic by dropping or marking the packets for which we don’t have enough tokens in the bucket.

### Class Based Policing

#### **Single Rate Policing**

To configure single rate policing, use one of the following commands inside a class of a policy-map:

```
R(config)# police cir CIR [bc BC [be BE]] [ACTIONS]
R(config)# police CIR [BC [BE]] [ACTIONS]
! CIR - in bps
! BC, BE - in Bytes. Default BC=1500.
```

If Be is set to a value higher than 0, this is a Single Rate, Three Color Marker. Otherwise, when Be=0 we have a Single Rate, Two Color Marker.\
You can also specify the value for CIR as percentage of the interface bandwidth. In this case, you sepcify Bc and Be in ms:

```
R(config-pmap-c)# police cir percent CIR [bc BC ms [be BE ms]] [ACTIONS] 
```

You can configure the ACTIONS in the same line or after you define the policer. The second option allows you to define more than one actions for each type:

```
! One line:
R(config-pmap-c)# police cir CIR ... conform-action ACTION [exceed-action ACTION [violate-action ACTION]]]
! Multiple actions:
R(config-pmap-c)# police cir CIR ...
R(config-pmap-c-police)# conform-action ACTION
R(config-pmap-c-police)# exceed-action ACTION
R(config-pmap-c-police)# violate-action ACTION
```

Available actions are:

```
 
  drop                              drop packet
  set-clp-transmit                  set atm clp and send it
  set-discard-class-transmit        set discard-class and send it
  set-dscp-transmit                 set dscp and send it
  set-frde-transmit                 set FR DE and send it
  set-mpls-exp-imposition-transmit  set exp at tag imposition and send it
  set-mpls-exp-topmost-transmit     set exp on topmost label and send it
  set-prec-transmit                 rewrite packet precedence and send it
  set-qos-transmit                  set qos-group and send it
  transmit                          transmit packet
```

#### **Two Rate Policing**

To configure a “Two Rate, Three Color” policer, use the command:

```
R(config-pmap-c)# police cir CIR [bc BC] pir PIR [be BE] [ACTIONS]
! Default BC: 1500
```

You can also specify the value for CIR as percentage of the interface bandwidth. In this case, you sepcify Bc and Be in ms:

```
R(config-pmap-c)# police cir percent CIR [bc BC ms] pir percent PIR  [be BE ms]] [ACTIONS] 
```

ACTIONS can be implemented as before.

#### **Three Level Hierarchical Policer**

Using MQC, you can enable a hierarchical policing, up to level 3. You achieve this by calling a policy-maps inside another policy-map. The objective is to police one type of traffic (say all http traffic) to CIR1, but inside this class, further police other type of traffic (say all http traffic to server A) to CIR2, while letting the rest of the traffic (say all http traffic to servers B and C) to be limited only by CIR1.

```
! Define Level 3 policy with police
R(config)# policy-map LEVEL3
R(config-pmap)# class CLASS3
R(config-pmap-c)# police ...
! Define Level 2 policy with police, and call policy LEVEL3
R(config)# policy-map LEVEL2
R(config-pmap)# class CLASS2
R(config-pmap-c)# police ...
R(config-pmap-c)# policy-map LEVEL3
! Define Level 2 policy with police, and call policy LEVEL3
R(config)# policy-map LEVEL1
R(config-pmap)# class CLASS1
R(config-pmap-c)# police ...
R(config-pmap-c)# policy-map LEVEL2
! Apply on the interface:
R(config-if)# service-policy {input|output} LEVEL1
```

The packets that are transmitted by the policer in LEVEL1 will enter the policer in LEVEL2. The same logic applies and packets that are transmitted by the policer in LEVEL2 will enter the policer in LEVEL3. Only the packets that are sent by the policer at level 3 will be actually sent

### CAR – Committed Access Rate

CAR is another method of implementing policing of traffic. See [CAR 101](https://nyquist.eu/car-101/).

## Shaping

When shaping, the excess traffic is delayed until there are enough tokens to be sent. This results in the levelling of the quantity traffic that is sent. There are more ways to implement traffic shaping:

### Class-Based Traffic Shaping

Class Bases Traffic Shaping uses MQC to set Traffic shaping on each class. Shaping can be set to conform to CIR or to PIR:

```
R(config-pmap-c)# shape {average|peak} CIR [BC [BE]]
! average rate conforms to CIR
! peak rate conforms to PIR
```

To calculate PIR, the router uses the formula PIR=CIR(1+Bc/Be)

#### **Hierarchical CBWFQ**

When using Hierarchical MQC policies, you can only use CBWFQ in the child policy(LEVEL2) only if you used shaping in the parent policy(LEVEL1)

```
! Define LEVEL2 policy with CBWFQ
R(config)# policy-map LEVEL2
R(config-pmap)# class CLASS2
R(config-pamp-c)# bandwidth KBPS
! Define LEVEL1 policy with shaping
R(config)# policy-map LEVEL1
R(config-pamp)# class CLASS1
R(config-pmap-c)# shape KBPS
! Apply the policy
R(config-if)# service-policy output LEVEL1
```

#### **Class-Based Adaptive Traffic Shaping for Frame Relay**

Additional features are available for Frame Relay interfaces:\
On Frame Relay interfaces you can also enable the adaptive feature, which makes a device react to frames received with the BECN bit set. When such frames are received, the shaping rate is reduced gradually (by 25% for each consecutive BECN) until it reaches a lower bit-rate defined with:

```
R(config-pmap-c)# shape adaptive LOW-RATE
```

The interface where the policy is applied can also be configured to send frames with the BECN bit set when it receives FECNs:

```
R(config-pma-c)# shape fecn-adapt
```

For best results, fecn-adapt and adaptive shaping should be configured on both ends of the connection.

### Generic Traffic Shaping

Generic Traffic Shaping is done on the interface. You can enable traffic shaping for all traffic, using:

```
R(config-if)# traffic-shape rate CIR [BC] [BE] [BUFFERS]
```

or you can enable traffic shaping only for packets that are matched by an ACL:

```
R(config-if)# traffic-shape group ACL CIR [BC] [BE] [BUFFERS]
```

To verify, use:

```
R# show traffic-shape [queue|statistics] [INTERFACE]
```

#### **Generic Adaptive Traffic Shaping for Frame Relay**

On Frame Relay interfaces you can also enable the adaptive feature, which makes a device react to frames received with the BECN bit set. When such frames are received, the shaping is reduced gradually (by 25% for each consecutive BECN) until it reaches a lower bit-rate defined with:

```
R(config-if)# traffic-shape adaptive LOW-RATE
```

The interface still needs to be configured with the traffic-shape rate command.\
The interface can also be configured to send frames with the BECN bit set when it receives FECNs

```
R(config-if)# traffic-shape fecn-adapt
```

For best results, fecn-adapt and adaptive shaping should be configured on both ends of the connection.\
To verify, use:

```
R# show traffic-shaping [statistics|queue] [INTERFACE]
```

### Legacy Frame Relay Traffic Shaping

See [Per VC Frame Relay QoS](../frame-relay-qos/#per-vc-frame-relay-qos)

### MQC for Legacy Frame Relay Traffic Shaping

[MQC Frame Relay Traffic Shaping](../frame-relay-qos/#generic-traffic-shaping) requires the use of a shaping policy-map configured with MQC that is used as the service-policy in a frame-relay map-class that is applied on an interface, subinterface or DLCI. You can only enable fragmentation at the interface level, but you can use adaptive traffic shaping commands in the MQC policy. You should not use the **frame-relay traffic-shaping** command.
