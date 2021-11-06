# Enhanced Object Tracking

Object tracking is an IOS feature that allows separation between the object that is tracked and the action to be tacken when the status of that object changes. The objects to be tracked are identified by a NUMBER and different client applications (HSRP, GLBP, VRRP) can track the staus of these objects and take appropriate actions.

## Define a tracked object

### Interface

```
R(config)# track NUMBER interface INTERFACE {line-protocol| ip routing}
```

You can track an interface status based on it’s line-protocol (up or down), or on it’s ip routing status. The ip routing status is considered up when all of the following are up:

* Interface line-protocol is up
* The interface has an IP address (static or dynamic)
* IP routing is enabled (on the interface)

### Route

```
R(config)# track NUMBER ip route PREFIX MASK {reachability| metric threshold}
R(config-track)# threshold metric up UP-TH [down DOWN-TH]
! only available for metric threshold.
R(config-track)# ip vrf VRF-NAME
! the route will be looked up in the routing table of vrf VRF-NAME
```

Tracking ip route reachability will only look for the existence of the PREFIX/MASK combination in the routing table, while metric threshold will consider the tracked object to be up if the scaled metric of the route is less then the UP-TH, and down if it is more than the DOWN-TH. The value of the threshold si 0-255, so the metric of the route will be scaled when compared.\
The value used for scaling is configurable with the following command:

```
R(config)# track resolution PROTOCOL VALUE
```

### IP SLA

```
R(config)# track NUMBER ip sla SLA-NUMBER {state|reachability}
! In older IOS versions:
R(config)# track NUMBER rtr SLA-NUMBER {state|reachability}
```

When tracking state, the object tracked will be “up” only if the IP SLA returns the code “OK”. All other return codes will translate to a “down” state. When tracking reachability, the object tracked will be “up” if the IP SLA returns the code “OK” or “over threshold”. All other codes translate to a “down state”

### Object Grouping

You can define a combinations of tracked objects, known as lists. A list will also have a status that can be tracked, but it’s status will be based on the status of other tracked objects.

```
! Boolean list
R(config)# track NUMBER list boolean {and|or}
R(config-track)# object CHILD-NUMBER [not]
```

A boolean AND object is up if all child objects are up. A boolean OR object is UP if at least one child object si up.

```
! Weighted list
R(config)# track NUMBER list threshold weight
R(config-track)# object CHILD-NUMBER [weight WEIGHT]
R(config-track)# threshold weight [up UP-TH|down DOWN-TH]
```

When a child object is up, it’s WEIGHT is added to the Parent’s Weight. The parent object will be up when it’s weight will be higher than UP-TH

```
! Percentage list
R(config)# track NUMBER list threshold percentage
R(config-track)# object CHILD-NUMBER
R(config-track)# threshold percentage [up UP-TH|down DOWN-TH]
```

The parent object will be up when a percentage higher than UP-TH is up.

### Delay

For all tracked objects you can define a timer that will delay the change of status:

```
R(config-track)# delay [up UP-DELAY|down DOWN-DELAY]
```

## Monitor tracked objects

```
R# show track [NUMBER]
R# show track brief
```
