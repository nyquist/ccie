# PPPoFR – PPP over Frame Relay

## PPPoFR

PPP can be used encapsulated in Frame Relay to offer all PPP features over an existing Frame Relay network. The first feature that comes to mind is probably authentication, but PPPoFR can also be used to offer advantages such as Multilink PPP over Frame Relay or PPP compression. Also, PPP can help in situations where point-to-point subinterfaces or inverse ARP cannot be used. PPPoFR will use IPCP instead of Frame Relay’s Inverse ARP to determine how to send an IP packet over the network. It also bypasses issues related to Split Horizons.\
In order to configure PPPoFR we will need to use a Virtual Template interface and a Virtual Access interface. The Virtual Template interface acts as a PPP interface and all configuration is done on it, but it will always show as down/down. The Virtual Access interface will get its configuration from the Virtual Template and will be the interface that will be up/up if everything works out.

To enable PPPoFR, follow these steps:\
1\. Enable Frame Relay encapsulation

```
R(config)# interface SERIAL-INT
R(config-if)# encapsulation frame-relay
```

2\. Enable PPP on the frame-relay interface:

```
R(config-if)# frame-relay interface-dlci DLCI ppp VIRTUAL-TEMPLATE-INT
```

3\. Then, configure the VIRTUAL-TEMPLATE interface with an IP address and with any other PPP options:

```
R(config)# interface VIRTUAL-TEMPLATE-IN
R(config-if)# ip address ...
R(config-if)# ppp ...
```

To verify, you can look at the routing table

```
R3#sh ip route
Gateway of last resort is not set

     2.0.0.0/32 is subnetted, 1 subnets
C       2.2.2.2 is directly connected, Virtual-Access1
     3.0.0.0/32 is subnetted, 1 subnets
C       3.3.3.3 is directly connected, Loopback0
```

As you can see, there’s the classic /32 ip route inserted by PPP, but it shows up connected on the Virtual-Access1 interface, not on Virtual-Template1. Also, look at the command below to see that the Virtual-Template interface is down/down while the Virtual-Access interface is up/up.

```
R3#sh ip int brief | i Virtual
Virtual-Access1            3.3.3.3         YES TFTP   up                    up
Virtual-Template1          3.3.3.3         YES TFTP   down                  down
Virtual-Access2            unassigned      YES unset  down                  down
```

## Ping yourself

To ping yourself in Frame Relay you must have a frame-relay map that points to your own IP. With PPPoFR it gets even more complicated, because the configuration uses a Virtual Acces interface that copies its configuration from the Virtual Template interface. Due to the fact that the Virtual Template interface is always down/down, ping to yourself will fail. The solution is to use ip unnumbered from a Loopback interface or make the virtual-template part of a PPP multilink interface.
