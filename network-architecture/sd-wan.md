# SD WAN

The enterprise version of Cisco SD WAN is based on Viptela and it is an overlay WAN infrastructure.&#x20;

## SD WAN components

### vManage NMS

This component is a single pane of glass NMS GUI that is used to configure and manage the SD WAN solution

### vSmart Controller

When a SD WAN router comes online it authenticates with the vSmart Controller. The vSmart Controller establishes a DTLS tunnel to each router. On top of these tunnels it establishes OMP (Overlay Management Protocol) neighborships with each SD WAN router.

OMP is a proprietary routing protocol that can advertise routes, next hops, keys and polisy information needed to maintain the SD WAN fabric.

Using OMP the vSmart Controller learns routes from SD WAN routers and calculates the best routes to network destinations. Then it advertises reachability information to all SD WAN routers in the fabric.

The vSmart Controller will also translate the policies defined in vManage NMS into a format supported by SD WAN routers and will push it to the devices.

### SD WAN routers

The SD WAN routers establish DTLS tunnels to vSmart Controllers and then form OMP neighborships with them. They also establish IPsec sessions with other SD-WAN routers in the fabric.

The SD WAN routers will make site-local decisions regarding routing, HA, interfaces, ARP management and ACLs.

There are 2 types of SD WAN routers:

* **vEdge** - original Viptela implementation
* **cEdge** - Viptela software integrated into Cisco IOS-XE

### vBond Orchestrator

The vBond Orchestrator authenticates the vSmart controllers and the SD WAN routers and orchestrates the connectivity between them. A vBond orchestrator has the following components:

* **Control plane connections**: Each vBond orcherstrator has a DTLS tunnel with each vSmart controller. It also uses DTLS to communicate with SD WAN routers to allow them to connect to teh SD-WAN fabric.
* **NAT traversal**: With this features the vBond Orchestrator facilitates connectivity between routers and vSmart controllers that are behind NAT&#x20;
* **Load Balancing**: When multiple vSmart Controllers are used, the vBond Orchestrator automatically performs load balancing of SD WAN routers across the vSmart controllers.

### vAnalytics

This is an optional component that provides analytics and service assurance.

## Cloud OnRamp

Cloud OnRamp is a set of functionalities that facilitates optimal connectivity for cloud SaaS applications or cloud IaaS environments.

