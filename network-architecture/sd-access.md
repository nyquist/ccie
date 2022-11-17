# SD Access

A fabric network is an overlay network built on top of the underlay network, the physical network. Fabric networks typically are built using tunneling technologies so thye can overcome limitations of traditional networks such as host mobility, lack of automation, difficulty of segmentation, difficulty of scalability.

Cisco SD Access is Cisco's campus network solution for an overlay fabric. It has 2 main components:&#x20;

* the fabric itself
* the solution to implement it and manage it. This solution can be:
  * Cisco DNA Center - and the ansamble is called Cisco SD Access
  * a combination of CLI or API calls using NETCONF/YANG - and the ansamble is called a campus fabric solution

## SD Access Physical Layer

The SD Access Physical Layer comprises of all network devices  that make up the network and the controller appliances

* Switches - provide LAN access to the fabric
* Routers - provide WAN and branch access to the fabric
* WLCs and WAPs - provide wireless access to the fabric
* Controllers: Cisco DNA Center and Cisco ISE

## SD Access Network Layer

The SD Access Network Layer consists of the underlay and the overlay networks.

### Underlay

The underlay is the physical network that is transporting data packets between network devices for the Overlay.&#x20;

The design goals for the Underlay are performance, scalability and high availability.&#x20;

The Underlay can be implemented as a traditional L2 network with STP but it is not recommended. Instead, the recommended method is to use Layer 3 Routed acces with IS-IS, but other routing protocols work as well.

The Underlay can be deplyed using:

* Cisco DNA Center via it's LAN Automation feature. It will implement an IS-IS route access campus design. While this option reduces the administrative complexity, it also reduces the customization options
* Using your own manual or home-grown automated CLI or API based management. It provides more customization options and allows you to chose your preferred routing protocols as well as re-use older networks to act as underlay.

### Overlay

The overlay is a virtual network that interconnects all network devices and provides all the additional features of SD Access.&#x20;

Even for overlay you can use Cisco DNA Center or your home-grown solution (manual or automated) but in the latter case it is not an "SD Access" solution, but rather a "Campus Fabric" solution

#### Control Plane

The SD Access Control Plane is based on **LISP** (Locator/ID Separation Protocol). LISP is a protocol that provides mapping between an Endpoint ID (EID) and a routing locator (RLOC). This approach allows for the separation between Endpoint IP and the location in the network.

LISP simplifies routing by moving remote destination information to a centralized database - LISP MS (Map Server) - which is a control plane node. This allows a router to maintain only its local routes and to ask the LISP MS for the location of destination EIDs.

Therefore LISP provides host mobility, address-agnostic mapping, smaller routing tables and built-in network segmentation through VRF instances.

#### Data Plane

VXLAN (Virtual Extensible LAN) is used to provide the data plane of SD Acess. It is an encapsulation technique based on UDP/IP which means it can be routed through an underlay network and creates the overlay network.&#x20;

VXLAN was preferred over LISP encapsulation because it also supports the encapsulation of the original MAC-in-IP, which LISP doesn't support. With this feature VXLAN can support virtual L2 networks or virutal L3 networks in the overlay.

The VXLAN Encapsulation takes the original L2 frame (Original Ethernet Header + Original IP Header + Data) and adds a VXLAN Header which is then encapsulated inside UDP>IP>Ethernet (Outer Ethernet Header + Outer VXLAN IP Header + Outer VXLAN UDP Header)

<figure><img src="../.gitbook/assets/vxlan_header.png" alt=""><figcaption><p>VXLAN Encapsulation</p></figcaption></figure>

| Field          | Description                                                                     |
| -------------- | ------------------------------------------------------------------------------- |
| R (Reserved)   | set to 0. 1 bit, part of the 8 bit Flags field                                  |
| I (Identifier) | set to 1 for a valid VNI                                                        |
| VNI            | identifies the individual VXLAN Overlay network where the communication belongs |
| Reserved       | Set to 0                                                                        |

Cisco also supports a modified VXLAN-GPO (Group Policy Option) header that supports TrustSec Scalable Group Tags (SGTs)

| Field                                |                                                                                                                                                                                                                                                                                                                           |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| G (Group Based Policy Extension Bit) | set to 1 to indicate an SGT tag is included in the header                                                                                                                                                                                                                                                                 |
| R (Reserved)                         | set to 0. 1 bit                                                                                                                                                                                                                                                                                                           |
| I (Indetifier)                       | set to 1 for a valid VNI                                                                                                                                                                                                                                                                                                  |
| D (Don't Learn Bit)                  | set to 1 to indicate that the egress Virtual Tunnel Endpoint (VTEP) must not learn the source address of the encapsulated frame                                                                                                                                                                                           |
| A (Policy Applied Bit)               | When the G bit is set to 1, the A bit indicates that the group policy hasn't already been applied when it is set to 0. This means that policy must be applied and after that the bit is set to 1. When the bit is already set to 1 it means the policy has already been applied and further policies must not be applied. |
| Group Policy ID                      | SGT tag                                                                                                                                                                                                                                                                                                                   |
| VNI                                  | identifies the individual VXLAN Overlay network where the communication belongs                                                                                                                                                                                                                                           |
| Reserved                             | Set to 0                                                                                                                                                                                                                                                                                                                  |

#### Policy Plane

The policy plane is based on Cisco TrustSec. With Cisco TrustSec, SGTs are assigned to authenticated groups of users or end devices. These SGTs can than be used for QoS, routing or network segmentation policies instead of using traditional L2 or L3 header fields.

#### SD Access Fabric Roles

<details>

<summary>Fabric Control Plane node</summary>

This node provides the EID-to-RLOC (Endpoint to Location) mapping so it is the LISP MS/MR (Map Server / Map Resolver)

The control plane receives registrations from Fabric Edge and Fabric Border nodes for known EID prefixes from wired or wireless endpoints.

It also replies to lookups from these nodes to locate destination EIDs

</details>

<details>

<summary>Fabric Border node</summary>

This node connects other L3 networks to the fabric. These nodes are LISP Proxy tunnel routers (PxTR)

There are 3 types of Fabric Border nodes:

* Internal Border: connects to known areas of the organization (for example firewall, data center)
* Default Border: connects to unknown areas outside of organization (e.g.: the Internet or public cloud). It is configred with a default route to reach these external unknown networks.
* Internal + Default Border: a combination of Internal and Default Border

</details>

<details>

<summary>Fabric Edge node</summary>

This node connects wired endpoints to the fabric. The edge node identifies and authenticates connected endpoints (through 802.1x). It then places them in a host pool (SVI and VRF) and assigns them to an SGT. Then it registere's the host's EID (L2 MAC, L4 IPv4 or L4 IPv6) with the control plane node.

The Fabric Edgen node provides a single L3 anycast gateway (same SVI with the same IP Address on all edge nodes).

This is also where the encapsulation/decapsulation of host traffic happens.

</details>

<details>

<summary>Fabric WLC</summary>

This node connects APs and wireless endpoints to the fabric. The WLC is external to the fabric and connects to the SD-Access through a Border Node. A fabric WLC also performs PxTR (Proxy Tunnel Routers) registration to the fabric control plane on behalf of the Fabric Edge

Traditionally the wireless traffic is encapsulated into CAPWAP to the WLC before it gets to the wired network. In SD-Access the data si distributed using VXLAN from fabric-enabled APs. A CAPWAP tunnel is still established and encapsualted into VXLAN for the control plane between AP and WLC.

</details>

<details>

<summary>Intermediate nodes</summary>

These nodes are used to connect the other nodes and only provide underlay services

</details>

#### SD Access Fabric Concepts

{% tabs %}
{% tab title="VN (Virtual Network)" %}
The VN provides the virtualization by using VRFs to create multiple L3 routing tables. In the control plane LISP instace IDs are used to maintain separate VRFs and in the data plane edge nodes add a VNI to the fabric encapsulation
{% endtab %}

{% tab title="Host Pool" %}
A host pool is a group of endpoints assigned to an IP pool subnet. Fabric Edgen nodes have an SVI for each host pool that will act as the default gateway for the hots.

Host pools can be assigned dynamically (using host authentication - 802.1x) or statically (per port).

The SD-Access fabric uses EID mappings to advertise each host pool. This allows host-specific (/32 IPv4, /64 IPv6 or MAC) advertisements and mobility.
{% endtab %}

{% tab title="Scalable Group" %}
A scalable group is a group of endpoints with similar policies. The SD-Access policy plane will assign each host to an SG using TrustSec SGT tags.

SG assignments can be static (per-port) or dynamic via 802.1x authentication

The SGT tags are added by the Fabric Edge or Fabric Border nodes ot the VXLAN header.
{% endtab %}

{% tab title="Anycast Gateway" %}
The anycast gateway provides a L3 default gateway where in such a way that the same SVI is provisioned on _every_ edge node withe the same SVI IP and MAC address. This allows for endpoint mobility which means an endpoint could connect to any edge node and still maintain it's reachability to the anycast gateway and to the memebers of the VN.
{% endtab %}
{% endtabs %}

## SD Access Controller Layer

The controller layer provides the management functions which are performed by Cisco DNA Center and Cisco ISE but it covers 3 subsystems:

* **Cisco NCP (Network Control Platform)** - integrated into Cisco DNA Center and provides the underlay and overlay automation and orchestration services
* **Cisco NDP (Network Data Platform)** - integrated into Cisco DNA Center and provies the data collection and analytics function as well as network assurance. It will provide the operational status of the network to the management layer
* **Cisco ISE (Identity Services Engine)** - provides NAC (Network Access Control) and identity services for dynamic endpoint-to-group mapping and policy definition. As part of the authentication process, ISE also provides the host pools and SG for the endpoints.

## SD Access Management Layer

The management layer provides the UI layer where information from all other layers are presented to the network operators. This layer abstracts a lot of the complex work performed by the other layers so it will provide a simplified view for operators who don't need to understand all complexities of the network. This layer is provided via Cisco DNA Center and tasks are implemented through a few workflows following the Intent-based networking paradigm:

* Cisco DNA Design Workflow
* Cisco DNA Policy Workflow
* Cisco DNA Provision Workflow
* Cisco DNA Assurance Workflow
