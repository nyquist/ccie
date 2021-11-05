# OSPF Areas

Areas are identified by a 32 bit Area ID. This can be represented as a number in decimal or in dotted decimal format. Area 0 (0.0.0.0) is reserved for backbone. The backbone is responsible for summarizing the topologies of each area to every other area => all inter-area traffic must pass through the backbone.

## Area Types

* **Normal** – Allowed LSAs: 1,2,3,4,5
* **Stub** – Allowed LSAs: 1,2,3 + Default Route as LSA 3 instead of LSAs 4 and 5
* **Totally Stubby** – Allowed LSAs: 1,2 + Default Route as LSA 3 instead of LSAs 3, 4 and 5
* **NSSA** – Allowed LSAs: 1,2,3 + LSA 7 for external routes from the local ASBR
* **NSSA Totally Stubby** – Allowed LSAs: 1,2 + Default route as LSA3 insted of LSAs3,4 and 5 + LSA 7 for external routes from the local ASBR

### Stub Area

* A stub area is an area into which LSA type 4 (ASBR Summary LSA) and 5(AS External LSA) are not flooded.
* As a resut, no external routes will be advertised into the area.
* Instead, ASBRs at the edge of the area use Type 3 LSAs to advertise a default route into the area
* LSAs allowed: 1,2,3 + Default Route instead of LSAs 4 and 5
* Routers configured for stub areas will not form adjacencies with routers not configured for stub ares
* Virtual Links cannot transit a Stub Area
* No router within a stub area can be an ASBR
* A Stub Area can have multiple ABRs but the routers inside cannot determine the best path to an ASBR (ABRs only advertise the default route)

```
R(config-router)# area AREA stub
R(config-router)# area AREA default-cost COST
! Assigns a specific cost to the default summary route used for stub areas
```

### Totally Stubby Area

* Uses a default route to reach AS external routes and inter-area routes
* ABRs only advertise a default route using a Type 3 LSA
* LSAs allowed: 1,2 + Default Route as LSA 3 instead of LSAs 3, 4 and 5(inter-area and as external routes)
* Just the ABRs will have to be configured with the no-summary option, the other routers can be configured only as stub

```
R(config-router)# area AREA stub no-summary
R(config-router)# area AREA default-cost COST
! Assigns a specific cost to the default summary route used for stub areas
```

### Not So Stubby Area (NSSA)

* Stubby Areas with an ASBR attached
* Since Type 5 LSAs are not allowed in Stub areas, the ASBR will originate a type 7 LSA to advertise external routes. This LSA flood stops at the ABR.
* The ABR will translate the Type 7 LSA into a Type 5 LSA to advertise it into area 0. If there are multiple ABRs, only one of them will be elected to translate LSA 7 into LSA 5 – the one with the highest Router ID
* ABR in a NSSA will not generate default routes, unless specified. If it injects a default route, then it will be sent as a Type 7 LSA
* LSAs allowed: 1,2,3,7

```
R(config-router)# area AREA nssa [no-redistribution] [default-information-originate] 
!default-information-origiante = ABR will insert a default route into the NSSA area as LSA type 7
!no-redistribution = ASBR will not insert external routes into the NSSA
```

Normally, NSSA Type 7 routes are redistributed into area 0 as Type 5 LSAs wich points the next hop to the ASBR that introduced the route. You can change this behavior with the following command:

```
R(config-router)# area AREA nssa translate type7 suppress-fa
```

The **translate type7 suppress-fa** keywords on the ABR will force it to translate Type-7 to Type-5 LSAs but change the next-hop to 0.0.0.0 when advertising into area 0. Heaving 0.0.0.0 in the Forward Address of an LSA means to use the advertising router’s address. Otherwise, the Type 5 LSA will have the ASBR’s address in the Forward Address field, which may be unreachable from other areas.

```
ASBRs can also summarize the routes inserted into OSPF with the command:
R(config-router)# summary address PREFIX MASK [not-advertise] [tag TAG]
!used on the ASBR to insert a summary route as LSA 7
```

### NSSA Totally Stubby

* The ABRs use a Type 3 LSA to advertise a default route instead of all other LSA types 3,4 and 5
* The Area also has an ASBR attached that advertises Type 7 LSAs
* ABR in a totally NSSA will generate default information by default into the area as LSA Type 3
* LSAs allowed: 1,2,7 + Default route using LSA 3 from ABR

```
R(config-router)# area AREA nssa no-summary 
```
