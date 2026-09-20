---
title: "Workaround for WSL1 DNS issue"
description: "Prevent WSL from overwriting DNS settings by setting generateResolvConf = false in..."
pubDatetime: 2026-01-19T11:30:31.752Z
---

* **Prevent WSL from overwriting DNS settings** by setting `generateResolvConf = false` in `/etc/wsl.conf`.
* **Pull active DNS servers from Windows** (based on default IPv4/IPv6 routes and interface metrics) using PowerShell, ensuring the most relevant adapters are used.
* **Write a stable Linux resolver configuration** by converting the Windows DNS list into `nameserver` entries and saving it to `/etc/resolv.conf` (with CRLF normalization via `tr -d '\r'`).

```bash
# Windows
wsl -d {distro}

# WSL
echo -e '[network]\ngenerateResolvConf = false' >> /etc/wsl.conf
exit

# Windows
wsl -t {distro}
wsl -d {distro}

# WSL
rm /etc/resolv.conf
/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe -NoProfile -Command '$ifs=@(Get-NetRoute -DestinationPrefix "0.0.0.0/0","::/0" -ErrorAction SilentlyContinue | Sort-Object RouteMetric,InterfaceMetric | Select-Object -ExpandProperty InterfaceIndex -Unique); $dns=foreach($i in $ifs){ (Get-DnsClientServerAddress -InterfaceIndex $i).ServerAddresses }; $dns | Where-Object { $_ } | Select-Object -Unique | ForEach-Object { "nameserver $_" }' | tr -d '\r' > /etc/resolv.conf
```
