# IS-IS 101

## Starting the routing process

Starting IS-IS process requires a 2 step configuration:\
1\. In the global config

```
R(config)# router isis [AREA-TAG]
!AREA-TAGs are used to run multiple IS-IS processes. Default: NULL
R(config-router)# net NETWORK-ENTITY-TITLE
!NETWORK-ENTITY-TITLE is in NSAP format. E.g 49.0001.0010.0100.1001.00
```

2\. On the interfaces that will be enabled for IS-IS

```
R(config)# interface INTERFACE
R(config-if)# ip router isis
```

## Passive interface

The passive interface command in IS-IS has a basically an opposite meaning to what it means in the other routing protocols. In IS-IS, since you have to manually select the interfaces that will run IS-IS and will send packets to form adjacencies, if you don’t want to run IS-IS on an interface you shouldn’t enable IS-IS on it. But what if you want to advertise it, without making any adjacency on it? Then, it’s a passive interface so go ahead and configure it as such 🙂

```
R(config-router)# passive-interface {INTERFACE-ID|default}
default: all local interfaces will be advertised
```

### Router Levels

By default Cisco IS-IS routers run at both Level1 and Level2. You can change the level on a per interface basis, using the command:

```
R(config-if)# isis circuit-type [level-1|level-1-2|level-2-only]
!Default: level-1-2
```

## Neighbors

Adjacencies are formed through the exchange of HELLO packets. On broadcast interfaces, there are separate HELLOs for each level, but on Point-to-Point interfaces, there is a single L1L2HELLO for efficiency.

### Authentication

Since HELLOs (ILH) are exchanged between neighbors and are not forwarded to other devices and since the packets describing the routes are forwarded to other ISes in the area or domain, there is a different mechanism of authentication for each type of packet:

#### **ILH Authentication**

Authentication for ILHs is done at the interface level:

```
R(config-if)# isis authentication mode {text|md5} [level-1|level2]
! if level is not selected, it applies to both levels.
R(config-if)# isis authentication key-chain KEY-CHAIN [level-1|level-2]
```

On older implementations, use:

```
R(config-if)# isis password PASSWORD
```

#### **LSP, CSNP, PSNP Authentication**

Authentication for these packets needs to be the same in the entire area, so this is done inside the routing process configuration:

```
R(config-router)# authentication mode {text|md5} [level-1|level-2]
! if level is not selected, it applies to both levels.
R(config-router)# authentication key-chain KEY-CHAIN [level-1|level-2]
```

On older implementations, use:

```
R(config-router)# area-password PASSWORD
! applies to Level1
R(config-router)# domain-password PASSWORD
! applies to Level2
```

## Timers

HELLO packets are sent every HELLO-INTERVAL – default 10 sec. The timeout value is based on the HELLO-INTERVAL and a HELLO-MULTIPLIER – default 10×3 = 30 sec. To change thse values, use:

```
R(config-if)# isis hello-interval SEC [level]
! default SEC = 10
R(config-if)# isis hello-multiplier N [level]
! default N = 3
```

However, once an IS is selected as the DIS for an area, it sends Hellos 3 times as fast (10/3 seconds) and has a similar hold value (30/3=10 sec) in order to detect failed DIS quicker.

## Packets

Hello Packets are exchanged every HELLO-INTERVAL (default: 10 sec) in order to create and maintain adjacencies and electing a DIS (similar to OSPF DR). On broadcast interfaces, separate HELLOs (IIH = IS-IS HELLOs) are sent for each level, while on point-to-point interfaces a single IIH is sent for both levels.\
LSP (Link State PDUs) are used to advertise the routing information. LSPs have variable size and include routing information as TLV (type, lenght, value) records inside the LSP.\
CSNP (Complete Sequence Numbers PDU) and PSNP (Partial Sequence Numbers PDU) are packets used to synchronize Link-state database.

## Metric

IS-IS can support several types of metrics, but only the “Default” metric is required to be implemented. It is normally associated with the circuit bandwidth. If other metrics are supported (Delay, Expense, Error) then a new SPF tree is created for each of them. Cisco routers usually support only the Default metric, but each circuit (interface) has a metric of 10, regardless of the bandwidth of that link. It’s up to the admin to change the metrics on the interface with the command:

```
R(config-if)# isis metric DEFAULT-METRIC [DELAY-METRIC] [EXPENSE-METRIC] [ERROR-METRIC] [level-1|level-2]
! if no level is specified, it applies to all levels.
```

Newer versions of the IOS use the Wide Metrics where values can are stored on 24 bits for individual metrics and on 32 bits for cumulative metrics. On older IOS versions, individual metrics had values between 1-63 (6 bits) and cumulative metrics had values between 1-1023 (10 bits).

## Adminsitrative Distance

IS-IS has an AD of 115.
