# Wireless Roaming

## Client perspective

A wireless client decides to roam to a different AP when the connection to the current AP si degraded. The roaming decision is entirely on the client side and can be caused by:

* maximum retries exceeded: Each vendor has a different threshold. One threshold could trigger a shift to a lower data rate and another threshold coudl trigger roaming
* Low RSSI
* Low SRN
* Proprietary roaming parameters - In some scenarios the APs or the controllers can communicate with the clients to trigger a roaming

In order to roam, a client needs to know about other APs providing access to the same SSID. To do this, the client needs to "scan":

* Active scan: The client changes its radio to a new channel and broadcasts a probe request. It usually waits 10ms for any responses.
  * Directed probe: The probe is sent for a specific SSID
  * Broadcast probe: The probe is sent to null SSID and all APs should respond with the SSIDs they support
* Passive scan: the client changes its radio to a new channle and waits for a periodic beacon. It usually waits for 100ms. Due to the longer wait time most clients prefer Active scanning

During a channel scan the client is unable to transmit or receive data. To reduce the impacts clients can do:

* Background scanning: Scannig happens only when the client is not transmitting or periodically on a single alternate channel to minimze data loss. This way, the client builds knowledge of available APs and can roam faster when needed.
* On-roam scanning: This occurs when roaming is necessary

## AP/Controller perspective

A Mobility Group (MG)is a collection of Mobiliy Controllers (MCs) accross which romaing needs to be supported. An MC can contain up to 24 WLCs. WLCs in a mobility group forward data traffic among the group which enables romaing between controllers and WLC redundancy.

Romaing inside a Mobility Group is done without the need to reauthenticate. If the client roams to an AP in a different WLC but in the same MG, the client datta will be transfered between WLCs.

Up to 3 MGs can be grouped in a Mobility Domain (MD) that supports up to 72 controllers. Clients can roam between controllers in differetn MGs as long as they are in the same MD.

If a client moves from one WLC to another one in the same MD the client needs to reauthenticate, reassociate and to get a new IP.

Controllers in an MG musth share a few parameters:

* Mobility Domain name
* Version
* CAPWAP mode
* ACLs
* WLANs (SSIDs)

WLCs sned mobility control messages between them using UDP 16666 (unencrypted). User data traffic is transmited using EoIP (IP protocol 97) or CAPWAP (UDP 5246) tunnels.

When a client associates and authenticate to an AP, the controller places an entry for the client in its database. This includes:

* MAC and IP address
* Security context and associations
* QoS contecxts
* SSID (WLAN)
* associated AP

## Types of Roaming

### L2 Roaming

L2 roaming occurs when the client moves from one AP to another but remains in the same subnet.&#x20;

* If the client roams from AP1 to AP2 but ends up on the same WLC, then we have **Intracontroller Roaming.** In this case the controller updates the database with the client's new AP.
* If the client roams from AP1 to AP2 but ends up on a different WLC, then we have **Intercontroller Roaming.** In this case the controlles exchanges mobility messages and client data is copied to the new controller. Intercontroller Roaming should remain transparent to the user unless the session timeout is exceeded or the client sends a DHCP Discover. In this case, POP and PoA move from old WLC to the new WLC.

### L3 roaming

L3 roaming occurs when the client moves from one AP to another and doesn't remain in the same subnet. This means the controller changed so it is an **Intercontroller Roaming.** But in this case, instead of moving the client DB to the new controller, the original controller marks the client with an anchor entry in it's own database.  The DB entry is copeid to the new controller  and marked as a foreign entry. The roam remains transparent to the client which gets to keep it's IP Address. This implies both anchro and foreign controller should have similar network access privileges so the client doesn't have connectivity issues after handoff. In this case the POP remains with the original WLC and the PoA moves to the foreign WLC.&#x20;

### Guest Tunneling (Auto-anchor mobility)

In this scenario you have one WLAN (ususally the Guest WLAN) that is tunneled to a predefined set of controllers to restrict clients to a specific subnet.&#x20;





