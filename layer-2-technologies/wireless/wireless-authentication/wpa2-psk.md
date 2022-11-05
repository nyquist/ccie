# WPA2 PSK

PSK authentication uses a symmetric encryption which means that the same key and algorrithm used to encrypt the message is used to decrypt it as well.&#x20;

An 802.11 WLAN client will use Open authentication by default. Open authentication uses no keys and doesn't offer end-to-end security. There is no encryption, per-packet authentication or message integrity check.&#x20;

PSK Authentication requires the key to have been shared with the AP and the client before the authentication process starts. The steps to authenticate using PSK are:

1. The client sends an **Authentication Request** to AP
2. The AP then sends a cleartext **challenge phrase** to the client
3. The client encrypts the phrase with the shared key and sends the **encrypted response** it back to the AP
4. The AP decrypts it with the shared key and checks if it matches the original challenge phrase
5. If the phrases match the AP sends an **Authentication Response** to AP
6. The client sends an **Association Request** to the AP
7. The AP sends an **Association Response** to the client
8. A **virtual port is opened** and the client data is now allowed
9. **Data** exchanged between client and AP will be **encrypted** using the same pre-shared key
