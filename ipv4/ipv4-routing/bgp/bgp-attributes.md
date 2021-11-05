# BGP Attributes

## BGP Data Structures

### Neighbor Table

```
R# show ip bgp neighbors [NEIGH-ADDR]
R# show ip bgp summary
```

The address in the bgp summary table shows the IP used in the peering, not the Router ID.

### BGP Table

Lists all prefixes learned from all peers

```
R# show ip bgp [topology]
! Status codes:
! > = Best route - will be installed in the routing table
! r = RIB failure:
R# show ip bpg rib-failures
```

If no routes towards a destination show the “>” code, you should investigate why no route is considered valid. Check with:

```
R# show ip bgp PREFIX
```

## Route Attributes

### Well Known

Well Known attributes are those attributes that must be supported by every BGP implementation.

#### **Mandatory**

Mandatory attributes are those attributes that must be sent in each Update.

{% tabs %}
{% tab title="AS_PATH" %}
Whenever a prefix is advertised to an eBGP peer, the AS is prepended to the AS\_PATH. AS\_PATH prepending is used to make the AS\_PATH longer than normal in order to make some routes more preferred than others.



**AS\_PATH prepending**

```
R(config-route-map)# set as-path prepend {AS-NUMBER AS-NUMBER... | last-as TIMES}
! Usually, the local AS is prepended multiple times
! When using last-as, the last AS, not the current AS is prepended several TIMES
```

**Ignoring AS\_PATH when searching best path**

The AS\_PATH is an important tiebreaker when selecting the best BGP route for a destination. However, it can be ignored with the hidden command:

```
R(config-router)# bgp bestpath as-path ignore
```

**AS\_PATH loop prevention**

AS\_PATH also acts as a Loop prevention mechanism. When a BGP peer receives a route that includes its own AS in the AS\_PATH, the update is dropped. An extreme way of filtering is to prepend the next AS to the AS\_PATH, forcing a neighbor to drop the route due to loop prevention.

However, there are situations when you would want to ignore this behavior and accept routes with your own AS-NUMBER in the AS\_PATH. (For example when you have the same AS number configured in 2 locations that connect over an ISP BGP cloud. Normally you would not accept routes that are originated in your own AS, but this time you should). To enable this feature, use:

```
R(config-router)# neighbor NEIGH-ADDR allowas-in TIMES
! accepts routes with the current AS in the AS-PATH. The AS may occur several TIMES in the AS_PATH
```

Another option si for the ISP to override the AS-NUMBER of the customer and replace it with its own AS in the AS-PATH. The command to do this is:

```
R(config-router)# neighbor NEIGH-ADDR as-override
! It will replace customer AS with the current router's AS
```

**Private AS**

When using Private ASes, you should strip them from the AS\_PATH when sending to an eBGP peer:

```
R(config-router)# neighbor NEIGH-ADDRESS remove-private-as
```

**Local AS**

Another way of modifying the AS\_PATH is via the **local-as** command, detailed [here](https://nyquist.eu/more-bgp/#5\_Local\_AS).

**AS\_PATH vs AS\_SET vs AS\_SEQUENCE vs AS\_CONFED\_SEQUENCE**

There are two types of AS\_PATH attributes:

* AS\_SET: an unordered list of AS numbers along a path. It is used when a BGP speaker creates an aggregate route. The AS\_SET will include all AS-es in the AS\_PATHs of the children routes in order to prevent loops. It will appear between {} in the AS\_PATH and will only count as 1 AS when calculating the shortest AS\_PATH. When the AS\_SET is included in the AS\_PATH, the ATOMIC\_AGGREGATE does not have to be included with the aggregate.
* AS\_SEQUENCE: an ordered list of AS numbers along a path
* When using confederations, an AS\_CONFED\_SEQUENCE appears in the AS\_PATH between (). They do not count in the Shortest AS\_PATH calculation.


{% endtab %}

{% tab title="ORIGIN" %}


Specifies the origin of the routing update:

* i = IGP – learned from an IGP. These routes are added to the BGP network using the “network” command
* e = EGP – learned from an EGP. You shouldn’t see this kind of routes in real life.
* ? = incomplete – usually as a result of redistribution. They are added to the BGP network using the “redistribute” command.

You can modify the origin of a route in a route-map, using:

```
R(config-route-map)# set origin {igp|egp AS-NUMBER|incomplete}
```
{% endtab %}

{% tab title="NEXT_HOP" %}


When advertising to an eBGP neighbor, the NEXT\_HOP is set to the advertising router’s interface address. When advertising to an iBGP peer, the NEXT\_HOP is not modified, unless the next-hop-self command is used:

```
R(config-router)# neighbor NEIGH-ADDRESS next-hop-self
```

You can change the next-hop of sent or received routes in a route-map used in the outgoing or the incoming direction.

```
R(config-route-map)# set ip next-hop IP1 [IP2...]
! Sets the IP address of the incoming/outgoing routes to the first reachable IP.
R(config-route-map)# set ip next-hop peer-address
! For incoming route-maps, sets the next-hop to the neighbor-address, regardless of the received value
! For outgoing route-maps, it is simialr to next-hop-self, but only applies to
```
{% endtab %}
{% endtabs %}

#### **Discretionary**

Discretionary attributes are those attributes that are supported by every implementation of BGP (Well Known) but may not be sent in every Updated

{% tabs %}
{% tab title="LOCAL_PREF" %}




Local Preference is used only in updates towards iBGP peers. If a BGP speaker receives multiple routes to the same destination, it compares the LOCAL\_PREF and the highest value wins. You can set the local preference in a route-map, using:

```
R(config-route-map)# set local-preference LOCAL_PREF
! default: 100
```

The default value of local-preference can be modified using:

```
R(config-router)# bgp default-local-preference LOCAL_PREF
! Default: 100
```
{% endtab %}

{% tab title="ATOMIC_AGGREGATE" %}
Used to alert downstream routers that a loss of path information has occured due to summarization. When the ATOMIC\_AGGREGATE is used, the BGP speaker can also use the AGGREGATOR attribute.
{% endtab %}
{% endtabs %}



### Optional

Optional attributes are those attributes that may not be supported by every BGP implementation.

#### **Transitive**

Transitive attributes are those attributes that may not be supported by every BGP implementation (Optional) but must be passed to the neighbor in the Update messages.

{% tabs %}
{% tab title="AGGREGATOR" %}
It contains the AS number and the Router ID of the router that originated the Aggregate route

****
{% endtab %}

{% tab title="COMMUNITY" %}
A community represents a group of destinations which share one or more common properties. Or, in other words, communities are labels that are attached to certain routes. The format of the COMMUNITY attribute is in the form of one 32 bit number (1-4.294.967.295). There is also a “new-format”, where the 32 bit number is split into 2×16 bit sub-values.

```
R(config)# ip bgp-community new-format
```

The default cisco format for community is AS:NN, where AS is the AS number where the community was originated. The default IEEE format is NN:AS.

* 0 – 65535 (0x000000 – 0x0000FFFF) – reserved
* User Defined
* Well Known
  * INTERNET – all routes belong to this community by default
  * NO\_EXPORT (0xFFFFFF01) – these routes cannot be advertised to eBGP peers or to peers outside the confederation
  * NO\_ADVERTISE (0xFFFFFF02) – routes received with this value cannot be advertised at all, neither to eBGP nor iBGP peers
  * LOCAL\_AS (0xFFFFFF03) or NO\_EXPORT\_SUBCONF – routes received with this value cannot be advertised to eBGP peers. If a configuration is configured, the routes cannot be advertised to other SUB-AS-es within the confederation

To attach a community to a route and send it to a neighbor:

1.  Use it in a route map:

    ```
    R(config-route-map)# set community COMMUNITY [additive]|COMMUNITY_LIST|none}
    ```
2.  Apply the route-map on a neighbor:

    ```
    R(config-router)# neighbor NEIGH-ADDR route-map ROUTE-MAP out
    ```
3.  Send community to the neighbor:

    ```
    R(config-router)# neighbor NEIGH-ADDR send-community
    ```

Once a community is set, it will overwrite all previous communities for that routes, unless the **additive** keyword is used.

A community list is used to match one or more Communities. To create a community-list, use:

*   Standard

    ```
    R(config)# ip community-list COMMUNITY-LIST {permit|deny} COMMUNITY
    ! COMMUNITY-LIST can be a string or a number (1 - 99)
    ```
*   Expanded

    ```
    R(config)# ip community-list COMMUNITY-LIST {permit|deny} REGEX
    ! COMMUNITY-LIST can be a string or a number (100 - 500)
    ```

You can match by a single community or by a Community list:

```
R(config-route-map)# match community COMMUNITY_LIST [exact-match]
! by default the COMMUNITY_LIST is matched using OR
! exact-match - use AND when matching
R(config-route-ma)# match community COMMUNITY
```
{% endtab %}
{% endtabs %}

****

#### **Non-Transitive**

Non-Transitive attributes are those attributes that may not be supported by every BGP implementation (Optional) are are not be passed to the neighbor in the Update messages.

{% tabs %}
{% tab title="MED (MULTI_EXIT_DISC)" %}


Allows an AS to inform another AS of the preferred ingress point. The lowest MED is preferred. The MED attribute is not passed beyond the receiving AS, so it is used to influence traffic between 2 directly connected AS-es. To influence route preference beyond the neighboring AS, use AS\_PATH prepend. If an ISP must forward traffic to a client AS, it will usually try to find the shortest way to send the traffic to the client AS. This behavior is cold “Hot Potato Routing”. However, the client might prefer this traffic to enter it’s network on a router that it’s closer to the actual destination. In this way, known as “Cold Potato Routing” the traffic will travel more inside the ISP and less inside the Client’s network. Routing by MED is one way of implementing “Cold Potato Routing”. By default, MED is compared only for routes that come from the same AS. Use the following command to compare MED regardless of the AS they were received:

```
R(config-router)# bgp always-compare-med
```

By default, routes are compared in order, the newest to the 2nd newest, the best of these to the third newest and so on to the oldest route. When comparing MED, this order can be changed using:

```
R(config-router)# bgp deterministic-med
```

When this command is enabled, the routes are grouped by the last AS and a best route is chosen for each group. if the “bgp always-compare-med” command is also used, the best routes for each group are compared between them and the lowest MED wins. Else, the MED will not be used because the routes came from different ASes and the route decision processes will go to the next step. See [this article](https://www.cisco.com/en/US/tech/tk365/technologies\_tech\_note09186a0080094925.shtml) for an example. By default, a Cisco router considers a route that doesn’t have the MED attribute set as heaving the value 0, which is the most preferred MED. You can change this behavior, using:

```
R(config)# bgp bestpath med missing-as-worst
```

This will make the router consider a route with a missing MED as the worst MED (4.294.967.295) You can enable MED comparison only for routes originated in the same confederation, using:

```
R(config-router)# bgp bestpath med-confed
```

When redistributing into BGP, the router will use the value set for the IGP metric as the value of the MED attribute of the route. If no value is set in the redistribution command, then the value of default-metric is used:

```
R(config-router)# default-metric MED
! Default: 0
```
{% endtab %}

{% tab title="ORIGINATOR_ID" %}
Contains the ROUTER\_ID of the originator of the route in the local AS (the last eBGP peer). A router should ignore routes received with its ROUTER\_ID as the ORIGINATOR\_ID. This attribute is set by the Route Reflector when reflecting routes to its clients.
{% endtab %}

{% tab title="CLUSTER_LIST" %}
A route reflector must prepend the local CLUSTER\_ID to the CLUSTER\_LIST. CLUSTER\_ID is the same as the ROUTER\_ID unless specifically changed:

```
R(config-router)# bgp cluster-id CLUSTER-ID
```

When a router receives a route with its Cluster ID in the Cluster List, the route is discarded.
{% endtab %}
{% endtabs %}

### Weight

It is a proprietary Cisco attribute that applies only to the current router and is not advertised to other peers. The higher the weight, the more preferable the route. By default, routes learned from peers have a weight of 0, while locally generated routes have a weight of 65535. To set the Weight of incoming routes, use:

```
! For a neighbor
R(config-router)# neighbor NEIGH-ADDR weight WEIGHT
! In a route-map
R(config-route-map)# set weight WEIGHT
```

## Choosing the best route

### Valid routes

In order to enter the route selection process, a route received via BGP must be considered valid. To do so, first it should be verified if the route appears in the BGP table:

```
R# show ip bgp
! * - Valid Route
! r - RIB Failure
! s - Suppressed
! d - Damped
```

Most often then not, the reasons why a route is not considered valid, are:

* If BGP Synchronization is enabled, the route is not found in the IGP table. When using OSPF, the Router ID of OSPF and BGP must match
* The router ignores the prefix because the current AS-NUMBER is already in the AS-PATH of the received route – eBGP only
* The route is dampened
* The next-hop is unreachable

For each prefix, the route that is chosen as the best route will be marked with the “>” character.

### Route Selection Process

1. **W** – Highest **W**eight – Cisco only
2. **L** – Highest **L**OCAL\_PREF
3. **OL** – Prefer a route that was **O**riginated **L**ocally. Routes originated with the network command are preferred to routes originated with the redistribute command, which are preferred to routes originated with the aggregate command, which are preferred to routes learned from other peers.
4. **A** – Shortest **A**S\_PATH
5. **O** – Lowest **O**rigin Code = i < e < ?
6. **M** – Lowest **M**ED if the routes go to the same AS
7. **E** – Prefer **e**BGP routes first, then iBGP routes
8. **N** – Shortest path to the **N**EXT\_HOP – lowest IGP (AD/metric) to the NEXT\_HOP
9.  **M** – If **M**ultipath is enabled, install multiple routes in the routing table, if they are from the same neighboring AS:

    ```
    R(config-router)# maximum-paths [ibgp] PATHS
    ! PATHS: 1 - 6. Default: 1
    ```

    Although multiple paths are added to the routing table, only one route will be considered best path and will be advertised to other neighbors. If Multipath is disabled, or no best route has been selected, then continue to the next step
10. **O** – Oldest routes – only for eBGP routes. There are situations when this step is skipped, like when using the command:

    ```
    R(config-router)# bgp bestpath compare-routerid
    ```

    See [this article](https://www.cisco.com/en/US/tech/tk365/technologies\_tech\_note09186a0080094431.shtml).
11. **R** – Prefer the route originated by the router with the lowest **R**outer ID (or Originator ID in Route Reflector environments).
12. **C** – For routes originated by the same Router ID, choose the one with the shortest **C**LUSTER\_LIST
13. **N** – Prefer the route that came from the **N**eighbor with the lowest address

Remember it as **With L.OL.A. O.M.E.N. M.O.R. C.N.**. For detailed information, read [this article](https://www.cisco.com/en/US/tech/tk365/technologies\_tech\_note09186a0080094431.shtml)
