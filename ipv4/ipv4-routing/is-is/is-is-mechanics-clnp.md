# IS-IS Mechanics – CLNP

## ISO OSI Terminology

| ISO OSI term        | TCP/IP Equivalent |
| ------------------- | ----------------- |
| End System          | Host              |
| Intermediate System | Router            |
| Circuit             | Interface         |
| Area                | Area              |
| Domain              | Autonomous System |

**IS-IS** = Intermediate System to Intermediate System\
**CLNP** = Connection-Less Network Protocol = Layer 3 network protocol that is used to communicate between ESes. CLNP offers a CLNS (Connection-Less Network Service). CLNP uses **NSAP** (Network Service Access Point) addressing. An NSAP address is assigned to an entire node, not to individual interfaces.\
**SNPA** = Sub-Network Point of Attachment = Layer 2 address\
**Local Circuit ID** = When added to the ISIS protocol, each interface gets Local Circuit ID. Values: 1-255.

### NSAP Format

The NSAP address has 2 parts, which can be split into smaller pieces:

* **IDP** = Initial Domain Part
  * **AFI** = Authority and Format Identifier = 1 octet long (00-FF). It identifies the format of the rest of the address. Usually 49 is used, which means the format/authority is local – similar to private addressing
  * **IFI** = Initial Domain Identifier = variable length, can miss if used only inside a domain
* **DSP** = Domain Specific Part
  * **HO-DSP** = High Order Domain Specific Part = variable length,identifies an area of the domain
  * **System ID** = Identifies the node in an area. Length can be between 1 and 8 Bytes. Usually 6 Bytes are used
  * **SEL** = Identifies the service that should process the packets on the node. If SEL=0, the frame is for the node itself and the NSAP address with SEL=0 is called **NET** = Network Entity Title

```
R(config-router)# net 49.0001.1111.1111.1111.00
! 49 = AFI
! 0001 = HO-DSP (Area)
! 1111.1111.1111.1111 = System ID
! 00 = SEL
```

## Routing Levels

**Level 0** = ES to ES on the same link (or ES to IS on the same link). ES and IS send Hellos – ESH and ISH – to advertise each other’s presence on the network.\
**Level 1** = ES to ES in a single area = IS nodes collect lists of ES nodes directly attached to them and exchange this information with all other IS in the area.\
**Level 2** = ES to ES in different areas from the same domain = IS nodes exchange area prefixes\
**Level 3** = ES to ES in different domains = Initially IDRP (Inter Domain Routing Protocol) was used to exchange domain information, but now BGP can be used since now it’s protocol independent.
