# Zone Based Firewall

## Basics

A Zone Based Firewall uses the same inspection engine as CBAC, but works with security zones, not with individual interfaces.\
A zone groups multiple interfaces together. By default, traffic is allowed between interfaces in the same zone, but is not allowed between interfaces in different zones. You can define zone pairs which are zones that can send traffic to each other.\
A special zone is automatically created for traffic destined to or generated by the router itself. This special zone called “self” allows traffic to/from it to the other interfaces.\
Zone Based Policy Firewall uses a syntax similar to [MQC used in QoS](https://nyquist.eu/qos-101-classifying-and-marking/#2\_MQC\_Modular\_QoS\_CLI)

## Configuration

### Define the Policy

```
! Create Class 
R(config)# class-map type inspect [mathc-any | match-all] CLASS
R(config-cmap)# match {access-group ACL | protocol PROTOCOL | class-map CHILD-CLASS}
! Create Policy
R(config)# policy-map type inspect POLICY
R(config-pmap)# class type inspect CLASS
! Define action:
R(config-pmap-c)# inspect PARAMETER-MAP
! Enables the CBAC engine
R(config-pmap-c)# police rate CIR burst BC
! Optional, sets policing
R(config-pmap-c)# drop [log]
! Drops the packets
R(config-pmap-c)# pass
! Allows packets to be forwarded
R(config-pmap-c)# urlfilter URL-PARAM-MAP
```

A parameter map is used to define the parameters of the inspection engine or of the urlfilter engine

```
! Define the parameter-map for inspection:
R(config)# parameter-map type inspect [PROTOCOL] {PARMETER-MAP | global | default}
! PROTOCOL specific options can be set inside the parameter-map
! > Start defining the Inspection parameters:
R(config-profile)# ?
! Define the parameter-map for URL Filtering:
R(config)# paramter-map type urlfilter URL-PARAM-NAME
! > Start defining the URL filtering params
R(config-profile)# ?
```

### Create the zones

```
R(config)# zone security ZONE-NAME
R(config-sec-zone)# description DESCRIPTION
```

### Add interface to a zone

```
R(config-if)# zone-member security ZONE-NAME
```

### Create Zone Pairs and attach policy

```
R(config)# zone-pair security ZONE-PAIR-NAME source {ZONE|self} destination {ZONE|self}
! Attach policy to zone
R(config-sec-zone-pair)# service-policy type inspect POLICY
```

Be aware that when you add an interface to a zone, by default, it will drop all traffic that is destined to other zones, unless there is a policy that permits it.\
One simple mistake is that the policy is only applied on one zone-pair (from interface A to interface B), but no policy is applied on the return direction (from interface B to interface A)\
This may not be harmful when you use Inspection, because inspected traffic will automatically be allowed on the return path, but if you use just **pass**, return traffic will be dropped if no service-policy that explicitly allows it is placed on the return path.
