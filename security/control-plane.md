# Control Plane

## CoPP – Control Plane Policing

Control Plane Policing is used to apply policy maps to traffic going to or coming from the control plane. This feature also mitigates DoS attacks by filtering traffic that arrives at the processor.\
First, you should define a [policy-map using MQC](https://nyquist.eu/qos-101-classifying-and-marking/#2\_MQC\_Modular\_QoS\_CLI). Then, apply this policy map to the control-plane:

```
R(config)# control-plane
R(config-cp)# service-policy {input|output} POLICY-MAP
```

To monitor Control Plan Policing use:

```
R# show policy-map control-plane
```

## CoPPr – Control Plane Protection

Control Plane Protection is similar to the policing feature, except it offers a more granular access to the control plane functions.\
You still need to define a [policy-map using MQC](https://nyquist.eu/qos-101-classifying-and-marking/#2\_MQC\_Modular\_QoS\_CLI). But now you can apply it on a sub-interface of the virtual control-plane interface:

```
R(config)# control-plane {host|transit|cef-exception}
! host - controls traffic that is directly destined for one of the router interfaces
! transit - controls traffic that is software-switched
! cef-exception - controls traffic that cannot be switched by CEF
R(config-cp-transit)# service-polcy input POLICY-MAP
```

### Layer 4 Port Protection

On the host subinterface you can use a special type of policy-map to deny access-to specific ports.\
First create the special class-map and policy-map:

```
R(config)# class-map type port-filter [match-all|match-any] PORT-FILTER-CLASS
R(config-cmap)# match [not] {closed-ports|port {tcp|udp} START-PORT [END-PORT]}
```

The **closed-ports** keyword should mean all ports that are not open on the control plane. Filtering traffic to the closed ports should spare the processor from unnecessary work.\
To see a list of open ports, use:

```
R# show control-plane host open-ports
```

Beware of the fact that some protocols (usually routing protocols) will not show up with open ports, so they should be specifically allowed if they are used.\
Then create the policy map:

```
R(config)# policy-map type port-filter PORT-FILTER-POLICY
R(config-pmap)# class PORT-FILTER-CLASS
R(config-pmap-c)# drop
```

In the end, apply it on the host subinterface:

```
R(config-cp-host)# service-policy type port-filter input PORT-FILTER-POLICY
```

### Queue Threshold Protection

Similar to the previous feature, you can set how long the queue of the control-plane host subinterface can be. Follow these steps to define the class-map and the policy-map:

```
R(config)#class-map type queue-threshold [match-all|match-any] QUEUE-TH-CLASS
R(config-cmap)# match [not] {protocol [PROTOCOL]|host-protocols}
! host-protocols - any open TCP/UDP port on the router
R(config)# policy-map type queue-threshold QUEUE-TH-POLICY
R(config-pmap)# class QUEUE-TH-CLASS
R(config-pmap-c)# queue-limit LIMIT
```

Then, apply it on the host subinterface:

```
R(config-pmap-host)# service-policy type queue-threshold input QUEUE-TH-POLICY
```

### Management Interfaces

Additionally, on the host subinterface, you can define what management protocols are allowed on each physical interface:

```
R(config-cp-host)# management-interface INTERFACE allow [PROTOCOL]
```

## Control Plane Logging

For traffic that arrives at the control plane (that is traffic that is not cef-switched), the control-plane can limit the amount of logging it does.\
To enable this feature, first define a logging class-map and policy-map:

```
R(config)# class-map type logging match-all LOG-CLASS
! Match by input interface:
R(config-cmap)# match [not] input-interface INTERFACE
! Match on source or destination address
R(config-cmap)# match [not] ipv4 {destination-address|source-address} IP-ADDR
! Match on packet action
R(config-cmap)# match [not] packets {dropped|error|permitted}
R(config)# policy-map type logging LOG-POLICY
R(config-pmap)# class LOG-CLASS
R(config-pmap-c)# log [interval SEC|total-length|ttl]
! interval - logs packets at the specified interval
! total-length - logs also packet length
! ttl - logs also ttl value
```

Apply the logging policy-map to the control plane interface or one of its subinterfaces:

```
R(config)# control-plane [host|transit|cef-exception]
R(config-cp)# service-policy type logging input LOG-POLICY
```
