# Routing Order of Operations

The original information was taken from Cisco article on [NAT Order of Operations](https://www.cisco.com/en/US/tech/tk648/tk361/technologies\_tech\_note09186a0080133ddd.shtml). However, this order helps understand other features, like WCCP.

1. If IPSec then check input access list
2. decryption – for CET (Cisco Encryption Technology) or IPSec
3. check input access list
4. check URPF (Unicast Reverse Path Forwarding)
5. check input rate limits
6. input accounting – update stats
7. redirect to web cache
8. If configured with ip nat outside: NAT outside to inside (global to local translation)
9. policy routing
10. routing
11. If configured with ip nat outside: NAT inside to outside (local to global translation)
12. crypto (check map and mark for encryption)
13. check output access list
14. inspect (Context-based Access Control (CBAC))
15. TCP intercept
16. encryption
17. Queueing
