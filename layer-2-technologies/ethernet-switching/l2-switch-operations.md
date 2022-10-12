# L2 Switch Operations

## CAM - Content Addressable Memory

A switch operates by forwarding frames based on the L2 MAC Destination Address. For each frame it receives, the switch looks up the destination address in the mac address-table and will find out on which port the destination is expected to be found.

The mac address-table is usually automatically built by the switch but it can also accept configurations to manipulate the mac address-table.

In order to build the mac address-table the switch uses the L2 MAC Source Address of a frame to update the table, recording the MAC address, the switchport the VLAN for the incoming frame and a timestamp of the arrival time. If the information already exists, only the timestamp is updated.

When looking up a Destination MAC address in the table there are 2 options

* an entry with the MAC address, port and VLAN exists in the table: In this case the frame is forwarded on the port.
* no entry is found: In this case the frame is forwarded to all ports in the same VLAN as the incoming port. This operation is also known as "Unknown Unicast Flooding"

The mac address-table is also known as CAM (Content Addressable Memory). A CAM works differently than a RAM (Random Access Memory). With RAM you can ask for a the content at a specific address, while with CAM you can ask for the address of a specific content.

To verify the contents of the mac address-table, you can use:

```bash
Sw# show mac address-table [interface INTF]
```

The mac address-table has an aging time. Each entry is kept in the table for until the aging time expires. By default this is set to 300 seconds (5 minutes) but it can be changed in config mode:

```bash
Sw# show mac address-table aging-time
Global Aging Time:  300
Vlan    Aging Time
----    ----------
Sw# conf t
Sw(config)# mac address-table aging-time SEC
# Use 0 to disable aging
```

## TCAM - Ternary Content Addressable Memory

Ternary CAM means this memory supports a third state as well, besides 0 and 1. The third state is X="don't care". This is implemented through a VMR format (Value, Mask, Result)&#x20;

The TCAM is used to hold security ACLs and QoS ACLs and frames would be tested against the TCAM entries to see if the frame should be sent or with what piority. TCAM is also used for L3 forwarding.&#x20;

The allocation of memory for TCAM tables is limited and statically allocated during the boot process but it can be slightly tweaked to use one of the possible SDM (Switching Database Manager) Templates.

```
Sw# show sdm prefer
...
Sw(config)# sdm prefer {vlan|advanced}
```

A reload will be required for the SDM templates to take effect.
