# NetFlow 102 – FNF – Flexible NetFlow

Netflow configuration is different by platform and IOS version. Initially, Cisco IOS supported what is now known as “Traditional/Original Netflow(TNF)”, but newer versions of the IOS support “Flexible Netflow (FNF). Support for Traditional Netflow configuration is being dropped from neweer IOS versions, so if available, use Flexible Netflow configuration on IOS and XR devies. Also, some devices support IPv6 flow monitoring only via FNF configuration. FNF and TNF can coexist on the same device. For more details about TNF, see [NetFlow 101 – TNF – Traditional NetFlow](netflow-101-tnf-traditional-netflow.md).

## How FNF works

FNF can be largely seen as a different way of configuring netflow. It is more “flexible” but in the end it accomplishes the same thing as TNF. FNF is enabled when a netflow monitor is applied on an interface, but there are a few prerequesits that need to be defined prior to this:

### Create NetFlow Record

#### **Pre-defined Flow Records**

Some IOS platforms have predefiend Flow Records. You can verify them with this command:

```
R#show flow record 
```

#### **Custom Flow Records**

On most platforms you can also define your own records:

```
R(config)#
!IPv4 record, similar to standard netflow v5 format
flow record FLOW-RECORD-IPV4
    match ipv4 tos
    match ipv4 protocol
    match ipv4 source address
    match ipv4 destination address
    match transport source-port
    match transport destination-port
    match interface input
    collect ipv4 dscp
    collect ipv4 ttl minimum
    collect ipv4 ttl maximum
    collect transport tcp flags
    collect interface output
    collect counter bytes
    collect counter packets
    collect timestamp sys-uptime first
    collect timestamp sys-uptime last

!IPv6 record, similar to standard netflow v5 format
flow record FLOW-RECORD-IPV6
    match ipv6 dscp
    match ipv6 protocol
    match ipv6 source address
    match ipv6 destination address
    match transport source-port
    match transport destination-port
    match interface input
    collect transport tcp flags
    collect interface output
    collect counter bytes
    collect counter packets
    collect timestamp sys-uptime first
    collect timestamp sys-uptime last
```

### Flow Exporter

```
!Define the exporter
R(config)# flow exporter FLOW-EXPORTER
!Configure the destination of the netflow traffic
R(config-flow-exporter)# destination NETFLOW-COLLECTOR-IP
!Configure the source of the netflow traffic
R(config-flow-exporter)# source INTERFACE
!Configure the port listening on the destination
R(config-flow-exporter)# transport udp PORT
!Configure the type of protocol used for tranport
R(config-flow-exporter)# export-protocol {netflow-v5|netflow-v9|ipfix}
!The following command will enable qos marking of the netflow packets based on output interface settings
R(config-flow-exporter)# output-features
```

### Flow Monitors

```
!Define the monitor
R(config)# flow monitor FLOW-MONITOR
!Configure the record type using custom or pre-defined records.
R(config-flow-monitor)# record {FLOW-RECORD-IPV4|FLOW-RECORD-IPV6|netflow ipv4 original-input}
!Tie the monitor to an exporter profile
R(config-flow-monitor)# exporter FLOW-EXPORTER
!Configure the caching parameters. Default: active = 60 sec, inactive: 15 sec.
R(config-flow-monitor)# cache timeout active SEC
R(config-flow-monitor)# cache timeout inactive SEC
```

If you want to enable netflow for both IPv4 and IPv6 you will need 2 different monitors, one for an IPv4 Flow Record, and one for an IPv6 Flow Record.

### Flow samplers (optional)

```
R(config)# sampler SAMPLER-1
! Random does random sampling. Deterministic does periodcal sampling (less overhead)
R(config-sampler)# mode {deterministic|random} 1 out-of WINDOW-SIZE
!Default: by packet
R(config-sampler)# granularity {connection|packet}
```

### Apply on the appropriate interface

```
!On each interface that should be enabled for Netflow caching:
R(config)# interface INTERFACE-NAME
R(config-if)# {ip|ipv6} flow monitor FLOW-RECORD [sampler SAMPLER-NAME] [multicast|unicast] {input|output}
!Without specifying unicast or multicast, the rotuer will performan netflow operations on both.
!The input/output will keyword will monitor incoming/outgoing traffic only on the interface.
!You will need to run the command twice (once in each direction) to perfrom netflow for incoming and outgoing traffic
```
