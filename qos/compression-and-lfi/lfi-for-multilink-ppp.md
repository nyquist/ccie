# LFI for MultiLink PPP

## LFI for MultiLink PPP on Serial Interfaces

PPP Multilink LFI (Link Fragmentation and Interleaving) allows a router to send big frames fragmented so that smaller, delay sensitive packets can be send between these fragments of larger frames. If LFI is not enabled, a packet would have to wait for the larger one to finish sending before it could be sent.\
By default, this feature is only available on Multilink PPP connections, not on individual PPP links. Also, this feature works by default only with pseudo-multilinks, that is multilinks that only use one link. This is because the smaller packets are not sent fragmented and if they came on different links it would confuse the receiving router.\
First enable [PPP Multilink](../../layer-2-technologies/layer-2-wan-protocols/ppp/ppp-multilink.md). Then add the LFI configuration on the multilink interface:

```
R(config-if)# ppp multilink interleave
```

When interleaving is on, the router will start fragmenting the packets. The size of a fragment is calculated as: FRAGMENT\_SIZE=INTERFACE\_BW\*FRAGMENT\_DELAY\
Modifying the interface bandwidth could affect other features, so it would be better to change the fragment delay in oreder to reach our fragment size goal. You can do this with:

```
R(config-if)# ppp multilink fragment delay MSEC
```

You can check the fragment size using:

```
R2#sh ppp multilink [active|interface MU1] | i frag size
    Se1/1, since 00:13:08, 386 weight, 378 frag size
```

Also, you should configure the interface with a queuing mechanism that would priorities the packets that we want to send between other fragments. For example a service policy with LLQ.

### Support for multiple links

You can enable support for LFI on Multilink interfaces with more than one link, if you enable ppp multiclass:

```
R(config-if)# ppp multilink multiclass
```

## LFI for MLPoFR

To enable LFI for Multilink PPP over Frame Relay links, you use the classic configuration with virtual-templates as seen [here](https://nyquist.eu/ppp-multilink/). The LFI configuration is done, same as before, on the Multilink interface.\
However, you should enable Frame relay Traffic Shaping on the physical interface:

```
R(config-if)# frame-relay traffic-shaping
```
