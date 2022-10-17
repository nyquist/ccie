# More EIGRP Features

## Router ID

The router ID is a 32 bit number, usually represented as a dotted decimal (like an IP address). The Router ID is determined when the routing process is started by the following algorithm:

1.  Use the configured value

    ```
    R(config-router)# eigrp router-id ROUTER-ID
    ```
2. Use the highest up/up loopback ip address
3. Use the highest up/up non-loopback ip address

The router ID should be different between neighbors. When a router receives a routing update from a router with the same Router ID, it will ignore the update considering it is from itself.

## DUAL

The following information can be found looking at the EIGRP topology table:

```
R# show ip eigrp topology
```

### Feasible Distance and the Successor

A router uses the EIGRP formula to calculate the metric for each destination for all available paths.\
For each destination, an EIGRP router calculates the metric based on information it has from the neighbors that advertise the destination and its incoming interfaces. See [EIGRP Metric](eigrp-metric.md). The best metric for each destination is called the **Feasible Distance**. This indicates the best path to reach the destination, and the router that is the next hop in this path is called the **Successor**.

### Reported Distance and Feasible Successors

For each destination, an EIGRP router also calculates a Reported Distance (RD), aka Advertised Distance, for the neighbor advertising it. It is actually the metric to the destination seen from the point of view of the neighbor. This will be lower than the metric calculated by the router.\
All routers that advertise a RD lower than the FD for each destination, are called **Feasible Successors**. These routers, are guaranteed to have a loop free path to the destination. EIGRP achieves quick conversion by immediately using one of the FS routes as the new Successor, in case the path via the Successor is down. All other paths cannot be guaranteed to be loop free and are not used in this step.

### Going Active – Sending Queries

What happens if there are no available FS in the topology table? The route will be marked as Active, and the router will send Queries to all its neighbors (except the Successor that just failed), asking if they have routes for that destination.

The neighbors will have to respond to the Query with a Reply, like this:

| Query from                       | Route state               | Action                                                                                                                                              |
| -------------------------------- | ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Successor                        | Passive                   | Attempt to find a new successor in the FS list. If none found, mark the destination active, and query all neighbors, except the previous successor. |
| Neighbor (not current Successor) | Passive                   | Reply with current Successor information                                                                                                            |
| Neighbor (not current Successor) | not in the topology table | Reply that the destination is unreachable                                                                                                           |

Waiting for each neighbor to reply, can put the route in a SIA (Stuck in Active) state. When a neghbor doesn’t reply in a resonable interval, a router will bring down the relationship with the neighbor. The default interval is 180 seconds and can be changed with:

```
R(config-router)# timers active-time {SEC|disabled}
```

To prevent unnecesary clearing of neighbor relationships, EIGRP sends at half the active-timer (90 sec by default) a SIA-Query message. If the router is still waiting for replies from its neighbors, it will send a SIA-Reply, so the active timer is reset. If the router does not receive a SIA-Reply, thent the neighbor relationship will be broken whent the active-timer expires.\
The Query domain can be limited by using route summarization and Stub Routers. The route will reply with an unreachable message if it has a route for a summary address, but not for the component route that he is beeing asked about.\
To see all current active routes, use:

```
R# show ip eigrp topology active
```

## Stub Routers

A stub router is a router that doesn’t need to know all routes in a network. It is usually the spoke in a hub-and-spoke network. Stub routers do not advertise EIGRP learned routes to other neighbors and Query messages are not sent to stub routers. To define an EIGRP router as stub, use:

```
R(config-router)#eigrp stub [receive-only | [connected] [redistributed] [static] [summary] [leak-map ROUTE-MAP]]
```

By default, if no option is used, IOS will use the **connected** and **summary** options.

* **receive-only** – does not advertise any routes
* **connected** – advertises connected routes but only for interfaces matched with a network command
* **summary** – advertises auto-summarized or statically configured summary routes
* **static** – advertises static routes, assuming the redistribute static command is configured
* **redistributed** – advertises redistributed routes, assuming redistribution is configured
* **leak-map** – allows a subset of the routing table to be advertised. This subset will be matched by the ROUTE-MAP

Usually, the upstream router of the stub router is configured to only send a summary or a default route to the stub router, but this has to be manually configured.

## Next Hop Self

By default, an EIGRP router will advertise routes with itself as the next hop. This can be changed per interface, with:

```
R(confi-if)#no ip next-hop-self eigrp AS-NUMBER
```

The command can be usefull in hub & spoke networks, when you want to have the spokes as the next hop instead of the hub.
