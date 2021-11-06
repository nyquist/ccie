# Classification and Marking

## Classification

### Classification using MQC

Classification is the process of assigning a packet to a category. QoS parameters are assigned to each category, therefore a different QoS level is applied for each packet. Classification of the packets can be done using information at Layer 2, Layer 3, Layer 4 or even Layer 7 (via NBAR). Classification is the first step in applying QoS policies and it can be done in several ways. Probably the most used method is via the MQC module of CLI.\
Inside a class-map define the match criteria using:

```
R(config-cmap)# match ATTRIBUTE VALUE
```

The attributes that can be used for a match include ACLs, other class-maps, source or destination MAC Addresses, Layer 3 DSCP, IP Precedence, Layer 2 CoS, etc:

```
R(config-cmap)#match ?
  access-group         Access group - match using ACL
  any                  Any packets
  class-map            Class map
  cos                  IEEE 802.1Q/ISL class of service/user priority values
  destination-address  Destination MAC address
  discard-class        Discard behavior identifier
  dscp                 Match DSCP in IP(v4) and IPv6 packets
  fr-de                Match on Frame-relay DE bit
  fr-dlci              Match on fr-dlci
  input-interface      Select an input interface to match
  ip                   IP specific values (DSCP values, IP Precedence, RTP Ports)
  mpls                 Multi Protocol Label Switching specific values (EXP)
  not                  Negate this match result
  packet               Layer 3 Packet length
  precedence           Match Precedence in IP(v4) and IPv6 packets
  protocol             Protocol
  qos-group            Qos-group
  source-address       Source MAC address
```

### Legacy methods of classification

Legacy methods of performing traffic classification are used by [CAR](https://nyquist.eu/car-101/), [CQ](https://nyquist.eu/legacy-congestion-management/#23\_Custom\_Queuing\_CQ) or [PQ](https://nyquist.eu/legacy-congestion-management/#24\_Priorty\_Queueing\_PQ).

### NBAR

NBAR is an advanced classification feature that is able to identify Layer 7 – Application traffic. To use NBAR in a class-map you have to use:

```
R(config-cmap)# match protocol PROTOCOL [OPTIONS]
```

NBAR has built-in support for most applications, but additional modules can be added either from a file:

```
R(config)# ip nbar pdlm PDLM-FILE
```

or by manual definition:

```
R(config)# ip nbar custom CUSTOM-NAME ...
```

#### **NBAR Protocol Discovery**

NBAR Protocol Discovery is a simple method of finding out what protocols run in the network. When enabled, it will start to account for the total number of bytes and packets for each type of protocol that is incoming or outgoing on the interface. It will also compute average bit rates. To enable NBAR Protocol Discovery, use:

```
R(config-if)# ip nbar protocol-discovery
```

To view the statistics, use:

```
R# show ip nbar protocol-discovery ...
```

## Marking

Usually, traffic classification is only performed at the network edge. Each packet will be considered as part of a class, and that class identifier will be inserted in the packet header. Marking a packet for QoS can be done at layer 2 or layer 3. Layer 2 markings can only be inserted if the encapsulation type permits, while Layer 3 markings use the IPv4 or IPv6 ToS Byte.

### Layer 2 Marking

On Ethernet links, a 3 bit CoS field (Class of Service) is encoded inside the 3 bits PRIORITY field of 802.1q, or inside the 4 bits USER field of ISL. This QoS information can only be used on trunk links, and there is no mechanism to carry it on access ports.

On Frame Relay links, we have 1 bit, called the “Discard Eligible” bit that can be used to mark traffic that exceeds the configured level of service.

MPLS also offers support for QoS markings inside its EXP field.

### Layer 3 Marking

For Layer 3 marking we use the Type of Service byte from the IPv4 header or the Traffic Class byte from the IPv6 header.\
Over the years, the interpretation of the IPv4 ToS byte has changed. First, only the 3 most significant bits were used for QoS. On 3 bits, only 8 classes could be encoded, and this was called IP Precedence. Later, the first 6 bits where used as the DSCP (Differentiated Services Code Point). The DSCP field was interpreted in such a way that was going to be backwards compatible with IP Precedence. A DSCP Class Selector has the same value for the 3 most significant bits as the IP Precedence.

#### **IP Precedence**

IP Precedence uses only the first 3 bits of the ToS field, so it offers support for only 8 markings. Actually, only 6 are available, since IP Precedence 6 (internet) and 7 (network) are reserved for network control protocols like routing updates.

| IP Precendece Bits | IP Precedence            | DSCP Bits | DSCP Class Selector |
| ------------------ | ------------------------ | --------- | ------------------- |
| 000                | 0 – Routine              | 000 000   | CS0                 |
| 001                | 1 – Priority             | 001 000   | CS1                 |
| 010                | 2 – Immediate            | 010 000   | CS2                 |
| 011                | 3 – Flash                | 011 000   | CS3                 |
| 100                | 4 – Flash Override       | 100 000   | CS4                 |
| 101                | 5 – Critical             | 101 000   | CS5                 |
| 110                | 6 – Internetwork Control | 110 000   | CS6                 |
| 111                | 7 – Network Control      | 111 000   | CS7                 |

#### **DSCP**

DSCP was introduced to allow a higher granularity of network traffic but also to provide compatibility with IP Precedence. Therefor the class system was used, where the available values are grouped into classes according to the first 3bits of the DSCP field. The values with 0 in the last 3 bits are called **CS (Class Selectors)** and they map to the 8 values of IP Precedence.\
**CS0** is the default DSCP value of a packet. It represents a DSCP field with all zeros (000000) and matches to IP Precedence 0.\
**CS1 through CS4** constitute the **AF (Assured Forwarding)** Classes. AF classes are further subdivided according to the Drop probability. These PHBs are noted AFxy. A higher x means traffic of higher priority, while a higher y represents traffic with a higher drop probability. AFxy is equivalent to a DSCP value of 8\*x + 2\*y. Bitwise, the x value is encoded in the 3 most significant bits of DSCP, while y is encoded in the next 2 bits. The last bit is always 0.\
When using DSCP,the highest priority PHB is **EF (Expedite Forwarding)** and it has a DSCP value of 46 (101 110)

| Drop probability | CS 0 | CS 1                              | CS 2                              | CS 3                              | CS 4                              | CS 5                            | CS 6 | CS 7 |
| ---------------- | ---- | --------------------------------- | --------------------------------- | --------------------------------- | --------------------------------- | ------------------------------- | ---- | ---- |
| Low              |      | <p>AF11<br>DSCP 10<br>001 010</p> | <p>AF21<br>DSCP 18<br>010 010</p> | <p>AF31<br>DSCP 26<br>011 010</p> | <p>AF41<br>DSCP 34<br>100 010</p> |                                 |      |      |
| Medium           |      | <p>AF12<br>DSCP 12<br>001 100</p> | <p>AF22<br>DSCP 20<br>010 100</p> | <p>AF32<br>DSCP 28<br>011 100</p> | <p>AF42<br>DSCP 36<br>100 100</p> |                                 |      |      |
| High             |      | <p>AF13<br>DSCP 14<br>001 110</p> | <p>AF23<br>DSCP 22<br>010 110</p> | <p>AF33<br>DSCP 30<br>011 110</p> | <p>AF43<br>DSCP 38<br>100 110</p> | <p>EF<br>DSCP 46<br>101 110</p> |      |      |

### Marking with MQC

Now, how do we configure a Cisco router to mark the packets? When selecting a class, inside a policy, we can instruct the router to mark the traffic with the appropriate PHB before sending out the packet. This is done using the set command:

```
R(config)# policy POLICY
R(config-pmap)# class CLASS
(config-pmap-c)# set ATTRIBUTE VALUE
!The attributes that can be set:
R(config-pmap-c)#set  ?
  atm-clp        Set ATM CLP bit to 1
  cos            Set IEEE 802.1Q/ISL class of service/user priority
  discard-class  Discard behavior identifier
  dscp           Set DSCP in IP(v4) and IPv6 packets
  fr-de          Set FR DE bit to 1
  ip             Set IP specific values (IP Precedence, DSCP)
  mpls           Set MPLS specific values (EXP)
  precedence     Set precedence in IP(v4) and IPv6 packets
  qos-group      Set QoS Group
```

When this policy is used on an interface, it will mark the packets that are matched by each class according to the ATTRIBUTES and VALUES used in the set command.

When setting some attributes like CoS, DSCP or IP Precedence, you can use a table map to configure an automatic conversion from one marking to another:

```
R(config)#table-map TABLE
R(config-tablemap)# map from VAL-FROM to VAL-TO
```

You can use this table with some **set** commands.

### Other ways to mark

There are other features that can be used to mark the packets, like [CAR](https://nyquist.eu/car-101/) or [Policing and Shaping](https://nyquist.eu/qos-101-policing-and-shaping/).
