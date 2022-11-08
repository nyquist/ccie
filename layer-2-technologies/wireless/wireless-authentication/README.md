# Wireless Authentication

Deprecated security standards:

* **WEP - Wired Equivalent Privacy** is weak and easily breakable so it is considered deprecated and shouldn't be used.
* **WPA - Wi-FI Protected Access** is also deprecated

Current security standards

* **WPA2** - successor of WPA. It implements the 802.11i security standard&#x20;

## WPA2

### WPA2 Personal mode (PSK)

With Personal mode, WPA uses a pre-shared key (PSK) that needs to be statically configured on client devices.

### WPA2 Enterprise mode (802.1X)

With Enterprise mode, [802.1X](wpa2-802.1x.md) and [EAP](../../../security/eap-101.md) are used for authentication so each user or device is individually authenticated

### WebAuth

With WebAuth guests can be authenticated in a secure way. WebAuth can also be used for clients that don't support 802.1X or clients that fail 802.1X authentication (as a backup mechanism)

### MAB (MAC Address Bypass)

MAB can be used for devices that don't support authentication that require user interaction. In this case the access to the network is allowed based on  the MAC Address of the device.

## WPS (Wi-Fi Protected Setups)

Some devices support the possibility of distributing stronger keys to the clients that want to connect. These methods usually require an admin to enter a small challenge phrase or to push a button (so it has physical access to the AP). These methods are not recommended as they have different weaknesses so WPS should be disabled.
