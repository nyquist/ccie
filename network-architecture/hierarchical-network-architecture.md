# Hierarchical Network Architecture

## Hierarchical LAN Design

When building a network it is a great advantage to follow a structured, hierarchical approach. The Hierarchical LAN Design breaks a complex networks into several layers that can provide improved resiliency, fault isolation and simplified maintenance.&#x20;

The Hierarchical LAN Design comprises of 3 layers: Access, Distribution and Core.&#x20;

At each aggregation layer the complexity is reduced from N\*(N-1) connections needed to mesh connect N devices down to 2\*N (when using individual links to a redundant aggregation devices) or even N (when using link aggregation to the aggregation devices).&#x20;

![Hierarchical LAN Design](<../.gitbook/assets/Network Architecture-3-tier.drawio.png>)

### **Access Layer**&#x20;

* Provides access to the network for endpoints and users. This means it also should support various technologies to accomodate different type of devices and network needs - for example, PoE (Power over Ethernet.
* It is the network boundary and therefore it acts as security, QoS and policy trust boundary. The access layer must protect the network from human errors and malicious attacks
* Traditionally access layer devices operate as L2 only devices but more modern takes on the design of the Access Layer will support devices operating at L3.

### **Distribution Layer**&#x20;

* aggregates the access layer and provides connectivity from one access layer device to another, or to the WAN or other services.&#x20;
* Traditionally access layer has devices operating at both L2 (towards the Access Layer) and L3 (towards the Core Layer).

### **Core Layer**&#x20;

* is used to aggregate multiple distributions. Without a core layer the distribution layer devices would need to be fully mashed.&#x20;
* It must be designed to provide packet swithing as fast as possible.&#x20;
* Devices of the Core Layer operate almost always as L3 devices.

## Hierarchical Design Options

### The 3 Tier Design

The 3 Tier design contains all three layers and makes use of the Core Layer to interconnect multiple distributions.

### Collapsed Core - The 2 Tier Design

This design usually works for smaller deployments where the effort of adding and maintaining a Core Layer to connect the distributions would not reduce the complexity enough. In a collapsed core design, the Core Layer functionality is provided by the distribution layer.&#x20;

Using the complexity formula to figure out when a collapsed core would be more beneficial than The 3 Tier Design, N\*(N-1) is higher than 2\*N when N is higher than 3.

![Collapsed Core Design](<../.gitbook/assets/Network Architecture-Collapsed Core.drawio.png>)

### Access Layer Design Options

* **Traditional Layer 2 Access** should be used with a Distribution Layer that makes use of a [FHRP](../ipv4/ipv4-addressing/fhrp-101.md) protocol to provide gateway redundancy for a VLAN.
  * Looped Design
    * In the looped design [STP ](../layer-2-technologies/ethernet-switching/802.1d-stp.md)must be used to break the loops. This will block traffic on some of the links thus making resource utilization sub-optimal
  * Loop Free Design
    * Is less common because it requires that each access switch supports a single VLAN which limits the flexibility of the network
* **Layer 3 Access**
  * This option moves the L2 boundry to the access switch, effectively avoiding the L2 loops and making use of all possible paths. However not all applications can work in a L3 only network and still require L2 connectivity between hosts.

### Simplified Distribution Layer

This design is called simpliefied because it makes use of technologies such as VSS (Virtual Switch System) or Stackwise.&#x20;

#### Using VSS

* **VSS** is usually used at the Distribution Layer and makes the (usually redundant) distribution devices work as a single logical device.&#x20;
* The Distribution Layer devices transfer control layer information over a Virtual Switch Link.&#x20;
* The design makes use of MEC (Multichassis EtherChannel) connections which bundles multiple physical links into a single logical link.&#x20;
* Since the VSS switch works as a single logical device there is no need for FHRP protocols for gateway redundancy.

#### Using Stackwise

* **Stackwise** can be used at the Access Layer and can connect multiple devices into a single logical link&#x20;
* The devices that are part of a stack exchange control plane information using the Stackwise Interconnect ring.
* The design makes use fo standard EtherChannel since the bundled links are between the same two devices.

![Traditional vs VSS vs Stackwise](<../.gitbook/assets/Network Architecture-VSS - Stackwise.png>)



<details>

<summary>Resources</summary>

* [Campus LAN and Wireless LAN Solution Design Guide](https://www.cisco.com/c/en/us/td/docs/solutions/CVD/Campus/cisco-campus-lan-wlan-design-guide.html)

</details>
