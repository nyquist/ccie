# OSPF Mechanics

## OSPF Router ID

Each router selects an OSPF Router ID when the OSPF process starts. The Router ID is a 32 bit number, usually written in dotted decimal format. The selection process is:

1.  Manually configured Router ID

    ```
    R(config-router)# router-id ROUTER-ID
    ```
2. Highest Loopback IP address
3. Highest non-Loopback “up/up” IP address

The interface used for Router ID doesn’t have to run OSPF and the router ID chosen when OSPF starts will remain even if the interface changes status or is deleted.\
To use a new Router ID, type:

```
R# clear ip ospf process
```

## OSFP Network Types

| Feature                   | Broadcast                     | Non-Broadcast      | Point-to-point             | Point-to-multipoint | Point-to-multipoint non-broadcast |
| ------------------------- | ----------------------------- | ------------------ | -------------------------- | ------------------- | --------------------------------- |
| Default for               | Ethernet                      | FR physical, FR MP | FR P2P, PPP, HDLC, Tunnels | –                   | –                                 |
| DR?                       | YES                           | YES                | NO                         | NO                  | NO                                |
| Hello Timer               | 10 sec                        | 30 sec             | 10 sec                     | 30 sec              | 30 sec                            |
| Dead Timer                | 40 sec                        | 120 sec            | 40 sec                     | 120 sec             | 120 sec                           |
| Hellos to                 | 224.0.0.5                     | Unicast            | 224.0.0.5                  | 224.0.0.5           | Unicast                           |
| Other packets to          | 224.0.0.5 (B)DR 224.0.0.6 DRO | Unicast            | 224.0.0.5                  | 224.0.0.5           | Unicast                           |
| Static Neighbor           | CAN                           | MUST               | NO                         | CAN                 | MUST                              |
| Multiple adjacencies      | YES                           | YES                | NO                         | YES                 | YES                               |
| Next hop for same segment | Advertising Router            | Advertising Router | Self                       | Self                | Self                              |
| Next hop for diff segment | Self                          | Self               | Self                       | Self                | Self                              |

* 224.0.0.5 is the All OSPF routers multicast address
* 224.0.0.6 is the OSFP DRs mulsticast address

In addition to these network types there is the stub network type that is the default for loopback interfaces. The stub networks will be advertised as /32 routes regardless of the interface mask. To advertise the network according to the mask, the network type should be changed to point-to-point.\
The network type is independent of the interface type, and the default values can be changed with:

```
R(config-if)# ip ospf network {broadcast|non-broadcast|point-to-point|point-to-multipoint [non-broadcast]}
```

To verify the network type, use:

```
R# show ip ospf interface INTERFACE
```

See this article about [running OSPF over Frame Relay](https://nyquist.eu/routing-over-frame-relay/#23\_OSPF).\
Routers can become neighbors even if the network type is different, as long as they agree on 2 things: If a DR is required or not, and if the HELLO/DEAD timers are the same. You can’t change weather a DR is required or not, but you can change the timers with the following commands:

```
R(config-if)# ip ospf hello-interval SEC
R(config-if)# ip ospf dead-interval SEC
```

To summarize, here’s what you need to remember:

| Keyword                                   | What it means                                                                | Network Type                                                                                           |
| ----------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| If it contains the word **POINT**         | <ul><li>No DR Election</li><li>Changes the next hop to self</li></ul>        | <ul><li>Point-to-point</li><li>Point-to-multipoint</li><li>Point-to-multipoint non-broadcast</li></ul> |
| If it contains the word **NON-BROADCAST** | <ul><li>Sends packets as unicast</li><li>Requires static neighbors</li></ul> | <ul><li>Non-Braodcast</li><li>Point-to-multipoint non-broadcast</li></ul>                              |
| If it is **usually used for Frame Relay** | <ul><li>Slow Timers</li></ul>                                                | <ul><li>Non-Braodcast</li><li>Point-to-multipoint</li><li>Point-to-multipoint non-broadcast</li></ul>  |

## Designated Router (DR) Election

When an OSPF router becomes active, it checks for an active DR and BDR on the networks that have a type that requires such a process (broadcast and non-broadcast). The router will wait for _WaitTimer_ (=RouterDead Interval) for a DR and BDR to be advertised in a Hello packet before starting an election process.

* If a DR and BDR exist, the router accepts them
* If there is no DR, but there is a BDR, the BDR becomes the DR and an election for BDR is held
* If there is no BDR and no DR, an election is held for both DR and BDR

If an election takes place, this is how the DR is chosen:

*   The router with the highest priority becomes the DR/BDR (depending on the election type)

    ```
    R(config-if)# ip ospf priority PRI
    !Default: 1
    !0 = never become a DR
    ```
* In case of a tie, the next criteria is highest Router ID

Since an existing DR and BDR is accepted, the first 2 routers that initialize on a Broadcast network will be selected as DR and BDR.

## Interface Status

* **Down**
* **Point-to-point** – The router sends Hellos and will attempt to establish an adjacency with the other end of the link. Option is available for Point-to-point, point-to-multipoint and virtual links
* **Waiting** – The router sends Hellos and waits for a Hello from DR and BDR. Option is available for Broadcast and NBMA networks
* **DR** – The router is the DR and will establish adjacencies with the DROthers
* **Backup** – The router is the BDR and will establish adjacencies with the DROthers
* **DROther** – The router is neither DR nor BDR and will establish adjacencies only with the DR and BDR, but will send Hellos to all routers
* **Loopback** – The interface is still advertised in Router LSAs even though packets cannot transit such an interfac

## Router Types

* **Internal** – All interfaces are in the same area
* **ABR (Area Border Router)** – Has at least one interface in area 0 and one in another area
* **Backbone Routers** – Routers with at least one interface in area 0
* **ASBR (AS Boundary Router** – Gateways for external trafic, injecting routes form other protocols into OSPF

## Virtual Links

Normally, traffic from one area to another must pass through Area 0. Sometimes, this is physically impossible, so the concept of virtual links was added. A virtual link can be used to create a neighbor adjacency for 2 routers that are not normally neighbors.

A virtual link extends area 0 from one ABR to another router in one of it’s non-zero areas. Therefore, the virtual link can only transit one area. A new adjacency will be formed over this virtual link so the area 0 extends to the other router, making it an ABR. The concept can be extended now, and a new virtual link can be created from this router to another router. Of course, the same rules apply.

Virtual Links cannot transit any flavor of stub areas.

To define a Virtual Link use the following command on the 2 ends of the virtual link. Of course, each router must reference the other router’s Router ID:

```
R(config-router)# area AREA virtual-link ROUTER-ID
```

If area 0 is set for authentication, then the virtual links must also be configured for authentication:

```
R(config-router)# area AREA virtual-link ROUTER authentication [message-digest]
! sets authentication as cleat text or MD5
R(config-router)# area AREA virtual-link ROUTER authentication-key CLEARTEXT-KEY
! sets clear text key
R(config-router)# area AREA_ID virtual-link ROUTER-ID message-digest-key KEY-ID md5 MD5-KEY
! sets md5 key
```

## OSPF over Demand Circuits

Periodic Hellos are suppresed and periodic refreshes of LSAs are not flooded. The circuit is used only at the initial db sync and only when changes have occured, in order to send the updated LSAs. Hellos are still sent over multi-access networks, but are not sent on point to multipoint network types.

```
R(config-if)# ip ospf-demand-circuit
```

Only one end of the Poin-to-pont connection or the multipoint in a Point-to-multipoint need this setting

## OSPF DNS Lookups

By default OSPF will not perform DNS lookup to translate the neighbor IP addresses to their hostnames. To enable, use:

```
R(config)#ip ospf name-lookup
```
