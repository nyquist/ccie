# EIGRP Metric

$$
Metric_{EIGRP} =\begin{cases}[(K_1B+\frac{K_2B}{256-L}+K3D)*{\frac{K_5}{R+K_4}}]*256 & K_5!=0\\ (K_1B+\frac{K_2B}{256-L}+K3D)*256 & K_5=0\\B+D & Default \end{cases}
$$

\
you will have to round down to the nearest integer

* default K values: **(K1,K2,K3,K4,K5) = (1,0,1,0,0)**
  * default EIGRP metric: **B+D**
  * remember it as **BuiLDeR 20**
* The EIGRP metric is 256 times larger than IGRP metric.

## Metric Components

* **Bandwidth – B**
  * inverse lowest bandwidth along the path in kbps, scaled by 10^7. The router receives the bandwidth "along the path" in advertisements from other routers. Therefore loweste bandwidth along the path will be the minimum value between the link's bandwidth and the advertised bandwidth.
  * Interfaces with bandwidth higher than 10Gbps (10^7 kbps) are considered similar from EIGRP’s metric standpoint (B=1). See Wide-Metrics if you need Bandwidth to account for interfaces over 10Gbps.

$$
B = \frac{10^7}{\min_{path}(Bw[kbps])} = \frac{10^7}{\min(Bw_{Link}, Bw_{Advertised})}
$$

* **Load – L**
  * Highest load along the path
  * Dynamically determined by the router. It has values starting from 1 (no usage) to 255(fully utilized)

$$
L = \max_{path}(Load) = \max(Load_{Link}, Load_{Advertised})
$$

* **Delay – D**
  * cumulative delay along the path in 10s of microseconds
  * EIGRP uses Delay to signal an unreachable route, by using the Delay value of 0xFFFFFF

$$
D = \sum_{path}(Delay[10 \mu s]) = Delay_{Link}+Delay_{Advertised}
$$

* **Reliability – R**
  * Lowest reliability along the path
  * Dynamically determined by the router as the percentage of successfully received packets on the interface scaled by 255. The maximum value of 255 means 100% reliability.

$$
R=\min_{path} (Reliability) = \min(Relability_{Link}, Reliability_{Advertised})
$$

* **MTU**
  * MTU is used as a tiebreaker if the metric is the same for more paths
  * Largest MTU wins the tiebreak

$$
MTU = \min_{path}(MTU_{interface})=\min(MTU_{Link}, MTU_{Advertised})
$$

* **Hop Count**
  * Not used in the actual metric, but the value is passed from router to router
  * There is a default limit of 100 to the number of hops to a destination, but this can be changed up to 255.

$$
Hop_{count}=\sum_{path}(Hops) = Hops_{Advertised} + 1
$$

When route information for a destination is received, it also contains the metric parameters used by the advertising router: Bandwidth, Delay, Load, Reliability, MTU. The receiving router compares the received information with the data it has for the incoming interface so it can find the lowest Bandwidth along the path, the highest Load along the path, the sum of Delays along the path, the lowest Reliability along the path and the lowest MTU along the path. Then, it can apply the formula to find the metric value for each path. The router that advertised the best path is considered the **Successor**, and the metric for that path is known as **„Feasible Distance” (FD)**

For each destination, the router also applies the formula on the parameters it received from the advertising router to calculate the metric from that router towards the destination. This is known as the **Advertised Distance (AD)** or **Reported Distance (RD).**

Using the DUAL algortithm, EIGRP will consider all paths that have a RD\<FD as loop free backup paths and will call the routers advertising them Feasable Successors(FS) for the route. The FS will be kept in the topology table and when the route through the successor fails, the router will imediately use the best FS. For all other routes, DUAL will not consider them as backup paths, because they can be part of a routing loop (even if they aren’t actually). The idea is that if RD>FD, then that router is closer to the destination then us and it should route through us. If we use it as our next hop then it could send it back to us resulting in a routing loop.

## How to modify metric

### Changing metric components

*   **Bandwidth**

    ```
    R(config-if)# bandwidth BANDWIDTH
    ! in kbps
    ```
*   **Delay**

    ```
    R(config-if)# delay DELAY
    ! in 10s of usec
    ```
*   **Load** and **Reliability** cannot be changed manually. They are caculated by the router based on a 5 minute average. The values can be seen using:

    ```
    R# show interface INTERFACE
    R3#sh int fa0/0 | i MTU|load
      MTU 1500 bytes, BW 10000 Kbit, DLY 1000 usec,
         reliability 255/255, txload 1/255, rxload 1/255
    ```

### Changing K values

```
R(config-router)# metric weights TOS K1 K2 K3 K4 K5
```

Default values for (K1,K2,K3,K4,K5) is (1,0,1,0,0) and only 0 is supported for TOS. K values must match between neighbors.

### Using offset lists

```
R(config-router)# offset-list ACL {in|out} OFFSET
```

Adds the OFFSET to the metric. It can be used to create multiple equal cost links.

## Maximum Hops

Even though EIGRP doesn’t use hop count in it’s metric calculation, it still counts how many hops away is the router that first advertised this network. You can see the hop count value using:

```
R1#show ip eigrp topology 4.4.4.0/24
IP-EIGRP (AS 10): Topology entry for 4.4.4.0/24
State is Passive, Query origin flag is 1, 1 Successor(s), FD is 2323456
Routing Descriptor Blocks:
24.0.0.2 (Serial1/0), from 24.0.0.2, Send flag is 0x0
Composite metric is (2323456/409600), Route is Internal
Vector metric:
Minimum bandwidth is 1544 Kbit
Total delay is 26000 microseconds
Reliability is 255/255
Load is 1/255
Minimum MTU is 1500
Hop count is 2
```

You can invalidate routes that are more than a number of hops away by running:

```
R(config-router)# metric maximum-hops HOPS
!Default: 100
```

All routes that have a hop-count greater than the maximum-hops, will not be added to the routing table.

## Wide metric

Newer IOS implementations support wide metrics, which can differentiate between interfaces higher than 10Gbps. [EIGRP Wide Metric](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute\_eigrp/configuration/xe-3s/ire-xe-3s-book/ire-wid-met.html) uses the following formula:

$$
WideMetric_{EIGRP} = (K_1B+\frac{K_2B}{256-L}+K_3D+K_6E)*\frac{K_5}{R+K_4}
$$



$$
Metric_{EIGRP} =\begin{cases}(K_1T+\frac{K_2T}{256-L}+K_3D+K_6E)*{\frac{K_5}{R+K_4}}& K_5!=0\\ K_1T+\frac{K_2T}{256-L}+K_3D+K_6E& K_5=0\\T+D & Default\end{cases}
$$

where&#x20;

* T = Throughput = $$\frac{10^7}{Bandwidth}*65536$$
* D = Latency = $$\begin{cases}\frac{Delay}{10}*65536 & if Bw < 1Gbps\\\frac{10^7*\frac{65536}{10}}{Bw} & if Bw >= 1Gbps\end{cases}$$
* E = Extended Attributes.&#x20;



See  for details. Supporting Wide metric, requires changes in the EIGRP packets to add the additional information. For this reason, a router will send packets for both standard metric and wide metric, so additional bandwidth will be used. If it detects that all neighbors on an interface support wide metrics, then it will only send this version.

When the wide metrics are used, the metric can become larger than the maximum value allowed by the RIB so it needs to be scaled down using the command&#x20;

```
R(config-eigrp)# metric rib-scale
```

When it is cofigured, all EIGRP routes in the RIB are cleared and replaced with the new metric values.

\
