# Switchport Port Security

Port Security restricts the number of stations that are allowed to access a switch port.

## Define allowed hosts

Each time a host attempts to send a frame, the source MAC address is added to the list of secure MACs. This list of secure MAC addresses has a limited size, and it can be configured with these types of secure MAC addresses:

*   **Static**: manually configured. Will appear in the running config.

    ```
    Sw(config-if)# switchport port-security mac-address MAC-ADDR
    ```
* **Dynamic**: learned from the source MAC of the frames that enter the port. They are not part of the config and will be lost on reload
*   **Sticky**: When enabled, it converts dynamic addresses into static sticky addresses so they will appear in the running config.

    ```
    Sw(config-if)# switchport port-security mac-address sticky
    ```

    . Static sticky addresses can also be added using:

    ```
    Sw(config-if)# switchport port-security mac-address sticky MAC-ADDR
    ```

## Enable port-security

To enable port security, the port must be statically set to access or trunk:

```
Sw(config-if)# switchport mode {access|trunk}
```

To enable port security, use:

```
Sw(config-if)# switchport port-security
```

By default, only 1 mac address will be allowed on this interface. You can modify the number of allowed mac addresses, using:

```
Sw(config-if)# switchport port-security maximum MAX
```

On trunk ports, the maximum value can be set per vlan:

```
Sw(config-if)# switchport port-security maximum MAX vlan [VLAN-LIST]
```

if VLAN-LIST is missing, then a maximum for each vlan is set.

## Security Violation

A security violation occurs when the maximum number of secure MAC addresses have been added to the list and a new station attempts to send a frame, or when an address learned or configured on one secure interface attempts to send a frame on another secure interface in the same VLAN.\
The default security violation mode is “shutdown” but this can be changed with:

```
Sw(config-if)# switchport port-security violation {protect|restrict|shutdown [vlan]}
```

* **protect**: Frames sent by hosts that are not in the secure list are dropped. No notification occurs
* **restrict**: Just like protect, but also sends a SNMP trap, a syslog message is logged and a violation counter increments
* **shutdown**: The interface is errdisabled. The switch sends a SNMP trap, a syslog message is logged and a violation counter increments
* **shutdown vlan**: Only the offending vlan is errdisabled. The switch sends a SNMP trap, a syslog message is logged and a violation counter increments

Recovery from the errdisabled mode can be manually (shut/no shut) or automatically:

```
Sw(config)# errdisable recovery cause psecure-violation 
```

## Clearing allowed hosts list

### Manual

The list of secure MAC addresses can be cleared manually, using:

```
Sw# clear port-security {all|configured|dynamic|sticky} [address MAC-ADDR|interface INTERFACE]
```

### Auto

The addresses in the list of secure MAC addresses can be aged out if you set:

```
Sw(config-if)# switchport port-security aging time SEC
```

The aging can be absolute (the address will be aged out after the configured interval) or relative to inactivity (the address will be aged out after an interval of inactivity equal to the configured value).

```
Sw(config-if)# switchport port-security aging type {absolute|inactivty}
```

Normally, the aging process happens only for dynamic addresses. It cannot be used for sticky addresses, but it can be used for static addresses, using:

```
Sw(config-if)# switchport port-security aging static
```

## Verification

To verify port-security status, you can use:

```
Sw# show port-security [address MAC-ADDR|interface INTEFACE]
```
