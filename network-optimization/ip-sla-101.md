# IP SLA 101

## About IP SLA

IP SLA is a feature that enables a router to monitor the status of a connection by measuring different KPIs. The SLA can be measured end-to-end from one host to another and is independent of the Layer 2 encapsulation.

## Configuring IP SLA

### Define the IP SLA Operation

First, you must define a SLA Operation:

```
R(config)# ip sla SLA-ID
```

Then, you must choose the SLA type:

```
R(config-ip-sla)# ?
IP SLAs entry configuration commands:
  dhcp         DHCP Operation
  dns          DNS Query Operation
  ftp          FTP Operation
  http         HTTP Operation
  icmp-echo    ICMP Echo Operation
  icmp-jitter  ICMP Jitter Operation
  path-echo    Path Discovered ICMP Echo Operation
  path-jitter  Path Discovered ICMP Jitter Operation
  tcp-connect  TCP Connect Operation
  udp-echo     UDP Echo Operation
  udp-jitter   UDP Jitter Operation
  voip         Voice Over IP Operation
```

Each type of SLA is doing a type of measuring and it has paritcular options that can be configured:

* **dhcp** – measures RTT (routing-trip-time) taken to discover a DHCP server and obtain a leased IP
* **dns** – measures RTT (routing-trip-time) taken to receive a DNS reply
* **ftp** – time taken to download a file from an FTP Server
* **http** – time taken to retrive a web page from an HTTP Server
* **icmp-echo** – measures end-to-end response time between the router and another IP device
* **icmp-jitter** – measures jitter, latency and packet-loss of the ICMP echos and echo-replies
* **path-echo** – measures end-to-end and hop-by-hop respone time
* **path-jitter** – measures end-to-end and hop-by-hop jitter, latency and packet-loss
* **path-jitter** – measures end-to-end and hop-by-hop jitter, latency and packet-loss
* **tcp-connect** – measures time to perform a TCP connect with a host
* **udp-echo** – measures end-to-end response when sending UDP packets
* **udp-jitter** – measures jitter when sending UDP packets. Useful for troubleshooting VoIP performance
* **voip** – voip related measurements

Once the operation type is selected, you can define even more options, like:

```
! Frequency of the operation
R(config-ip-sla-echo)# frequency SECONDS
! Timeout of the response:
R(config-ip-sla-echo)# timeout MSEC
! Limit of the rising threshold 
R(config-ip-sla-echo)# threshold MSEC
! Size of the Padding:
R(config-ip-sla-echo)# request-data-size BYTES
! ToS value of the packets sent
R(config-ip-sla-echo)# tos TOS-VALUE
! Force the router to check for data-corruption:
R(config-ip-sla-echo)# verify-data
! SNMP Owner
R(config-ip-sla-echo)# owner STRING
```

IP SLA maintains several history statistics. They can be configured with the history command:

```
R(config-ip-sla-echo)#history ?
  buckets-kept                      Maximum number of history buckets to collect
  distributions-of-statistics-kept  Maximum number of statistics distribution buckets to capture
  enhanced                          Enable enhanced history collection
  filter                            Add operation to History when...
  hours-of-statistics-kept          Maximum number of statistics hour groups to capture
  lives-kept                        Maximum number of history lives to collect
  statistics-distribution-interval  Statistics distribution interval size
```

### Schedule or start the operation

```
R(config)# ip sla schedule SLA-ID [start-time WHEN] [recurring] [life {SEC|forever}] [ageout SEC] 
! life = how long to run the SLA operation
! ageout = how long to keep the Entry when inactive
! recurring = reschedule it daily
! start-time: WHEN can be one of the following:
!       HH:MM:[SS] - start at the specified time
!       after HH:MM:SS - start HH hours, MM minuts, SS seconds later
!       now - start now
!       pending - does not collect information
```

You can restart a SLA operation using:

```
R(config)# ip sla restart SLA-ID
```

### Configure the responder, if needed

The responder should be configured on a router that responds to IP SLA requests

```
! TCP Connect or UDP Echo:
R(config)# ip sla responder {tcp-connect|udp-echo} IP-ADDR port PORT
! Frame-Relay:
R(config)# ip sla responder frame-relay all
```

A SLA responder and an initiator can be authenticated using a Key-chain:

```
R(config)# ip sla key-chain KEY-CHAIN
```

## Monitoring SLA

You can monitor the status of the SLA operations using:

```
R# show ip sla statistics SLA-ID [details]
! or some other show ip sla commands
R# show ip sla {configuration...|history...| ...}
```

### Proactive Monitoring

Proactive monitoring allows a router to take action when a SLA operation is below requirements. You can enable sending of trap messages using:

```
R(config)# ip sla logging traps
```

A reaction can be defined with the following command:

```
R(config)# ip sla reaction-configuration SLA-ID react EVENT ... 
! The reaction can be to send a trap, to generate a trigger, both or none
```

If a trap is generated then the SNMP server must be configured to send that kind of trap:

```
R(config)# snmp-server enable traps TRAP
```

If a trigger is generated, it can put into an active state other SLA operations:

```
R(config)# ip sla reaction-trigger SLA-ID TRIGGERED-SLA-ID
```
