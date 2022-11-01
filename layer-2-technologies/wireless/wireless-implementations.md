# Wireless Implementations

## Wireless Standards

{% tabs %}
{% tab title="802.11 a/b/g (Legacy)" %}
Year ratified:&#x20;

* 1999 (a/b)
* 2003 (g)

Frequency Band:&#x20;

* 5 GHz (a)
* 2.4 GHz (b,g)

Data Rates:&#x20;

* 11Mbps (b)
* 54Mbps (a,g)

Features:

* SISO
{% endtab %}

{% tab title="802.11n" %}
Year ratified:&#x20;

* 2009

Frequency Band:&#x20;

* 5 GHz
* 2.4 GHz

Data Rates:&#x20;

* Up to 600 Mbps (channel bonding for up to 40MHz)

Features:

* backwards compatible with 802.11a/b/g
* MIMO
{% endtab %}

{% tab title="802.11ac" %}
Year ratified:&#x20;

* 2013

Frequency Band:&#x20;

* 5 GHz&#x20;

Data Rates:&#x20;

* 1300 Mbps - Wave 1 (channel bonding of up to 80MHz)
* 6930 Mbps - Wave 2 (channel bonding of up to 160MHz)

Features:

* 802.11ac is backwords compatible with 802.11a and 802.11n
* MU-MIMO
{% endtab %}

{% tab title="802.11ax (Wi-Fi 6)" %}
Year ratified:&#x20;

* 2021

Frequency Band:&#x20;

* 5 GHz
* 2.4 GHz&#x20;

Data Rates:&#x20;

* 4800 - Wave 1
{% endtab %}
{% endtabs %}

**SISO** is a system where a system uses a single antena at a time even if it has multiple antennas. Systems that can use multiple antennas symultenousley are called **MIMO**. MIMO incorporates 3 technologies:

* **MRC (Maximal Ratio Combining)** - a MIMO receiver uses MRC to combine energies from multiple recive chains
* **Beamforming** - a MIMO transmitter can coordinate the signal sent from each antenna so that the receiver gets a better signal. Cisco ClientLink is a beamforming technology
* **Spatial Multiplexing** - requires a MIMO transmitter and a MIMO receiver and allows the transmitter to split the data in multiple streams and send them to each antenna of the receiver.

While these features improve communication between one sender and one receiver at a time, **802.11ac MU-MIMO** allows the AP to transmit frames to multuple clients at the same time.

## Wireless Component Roles

### Clients and Access Points (APs)

An AP functions similarly to an Ethernet hun in that only one device can talk to the AP at a given time, over a shared media. A client's connection state to an AP can be one of:

* Not authenticated and not associated
* Authenticated but not associated (yet)
* Authenticated and associated - only in this state the data can flow

The associataion process has several steps:

1. mobile station sends a probe to discover available networks. Probe requests are sent to BSSID FF:FF:FF:FF:FF:FF (it will be received by all APs) and advertise the supported data rates and capabilities of the station
2. APs receving the probe request check to see if they support any of the advertised data rates and a probe response is sent with the SSID, supported data rates, encryption type and capabilities of the AP
3. Based on the responses received, the mobile station chooses a compatible network and sends an 802.11 authentication Open message with Seq set to 1(not the same authentication as WPA or 802.1x)
4. The AP receives the authentication frame and reponds with autehntication Open and Seq set to 2. (since authentication is open most requests should be succesful)
   1. If AP receives any frame other than an authentication or probe request from a station it will respond with Deauthentication frame and it will place the station in an "unauthenticated and unassociated state"
5. A station that received the Authentication Open with Seq=2 frame will send an association request to the AP.
   1. If AP receives any frame other than an assocation request from a station it will respond with Deassociation frame and it will place the station in an "authenticated but unassociated state"
6. If the asociation paramters match, the AP will create an Association ID and reply to the station with an Association response. At this point the client is authenticated and associated.

### Wireless controller

Enterprise solutions may require a large number of APs that would be difficult to adminsitrate and coordinate if they act as independent APs. For this reason there are solutions that make use of a Wireless Controller. Cisco's solution is called WLC (Wireless LAN Controller). In this case the functions of a traditional AP are split between the AP and the controller.

#### CAPWAP (Control and Provisioning of Wireless Access Points)

CAPWAP is an open protocol that enables a WLC to manage APs. The AP and WLC build a secure DTLS tunnel (control plane) to communicate. The client data is encapsulated with a CAPWAP header and is sent to the WLC.

#### Mobility Controller (MC) and Mobility Agent (MA)

MA and MC are functions running on WLC. MA is responsible to terminate CAPWAP tunnels so it maintains a cliend database while MC provides mobiloity management tasks including roaming, wireless IPS, guest access. MA reports local and roamed client states to MC.

#### POP and PoA FUnctions

The POP is the Point of Presence for the client. It anchors the client IP Address and is used for security policy applications. The PoA is the Point of Attachement. It moves with user AP connectivity and it is used for user mobility and QoS policy application.

Before a user roams the POP and PoA are the same but if the user roams the PoA may move as well.
