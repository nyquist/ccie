# IP RTP Priority

## IP RTP Reservation

IP RTP Reserve adds RTP traffic to the RSVP-reserved Queues of WFQ. These queues have a weight of 128, but RTP traffic will still have to compete with other RSVP flows for the bandwidth. Additionally, traffic that exceeds the configured RTP-BW will be treated as traffic with IP\_Priority 0.\
To configure IP RTP Reserve, use:

```
R(config-if)# ip rtp reserve UDP-START UDP-RANGE RTP-BW
```

The IP RTP Priority feature was designed to provide strict priority queuing for voice traffic or any other delay sensitive traffic that uses RTP. This is better than the IP RTP reserve feature, where RPT traffic still had to compete with RSVP flows for bandwidth.

IP RPT Priority is compatible with WFQ and CBWFQ and it is used on an interface. You will have to configure the range of ports used by the RTP application that is given priority and how much bandwidth is allocated to it.

```
R(config-if)# ip rtp priority UDP-START UDP-RANGE RTP-BW
```

Effectively, RTP Priority will assign RTP traffic to a Priority Queue that is given strict priority before WFQ or CBWFQ. Only even ports are matched by this command, in contrast to RTP Reservation. This is because usually Voice traffic uses even ports, while voice signaling uses odd ports.

Traffic is rate limited to the RTP-BW, just like in RTP Reservation. RTP Priority will only need as a parameter the bandwidth of the RTP packets (compressed or not), without the L2 overhead.

The IP RTP Priority feature will take precedence over all other traffic, including the LLQ priority class.

### IP RTP Priority for Frame Relay

You can configure IP RTP priority on frame-relay interfaces using a map-class. See [Frame Relay QoS](../frame-relay-qos/) on details on how to configure and apply frame-relay map-classes.\
To enable IP RTP in a map-class, use:

```
R(config-map-class)# frame-relay ip rtp priority UDP-START RANGE RTP-BW
```

Frame Relay Traffic Shaping (FRTS) and Frame Relay Fragmentation (FRF.12) must be configured for FR ip rtp priority to work.
