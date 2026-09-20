---
title: "Generate Reverse DNS (PTR) Names from IP Addresses via Python CLI"
description: "A short Python script converts one or more IPv4 or IPv6 command-line arguments into the corresponding reverse DNS query names using the standard ipaddress module."
pubDatetime: 2026-01-19T11:14:17.234Z
---

* Accepts one or more IP addresses as command-line arguments (`sys.argv[1:]`).
* Uses `ipaddress.ip_address()` to parse each IPv4/IPv6 address safely and consistently.
* Outputs the corresponding reverse DNS query name (`ip.reverse_pointer`), e.g., `in-addr.arpa` for IPv4 and `ip6.arpa` for IPv6.

```python
import ipaddress
import sys

if __name__ == "__main__":
    for arg in sys.argv[1:]:
        ip = ipaddress.ip_address(arg)
        print(ip.reverse_pointer)
```
