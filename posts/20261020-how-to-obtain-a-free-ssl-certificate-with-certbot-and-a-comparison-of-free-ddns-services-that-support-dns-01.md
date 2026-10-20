---
title: "How to Obtain a Free SSL Certificate with Certbot, and a Comparison of Free DDNS Services That Support DNS-01"
description: "Explains how to use Certbot's DNS-01 authentication to get wildcard certificates with free DDNS services, and compares No-IP, DuckDNS, Dynu, and dynv6."
pubDatetime: 2026-10-20T12:55:00+09:00
---

If you want to use HTTPS on your home server or VPS, you can obtain and renew SSL/TLS certificates for free using Let's Encrypt and Certbot.

However, you may encounter situations where you cannot open TCP port 80 on your home server, or you want to manage multiple subdomains.

In particular, free DDNS services like No-IP have restrictions on DNS record manipulation, which can prevent you from obtaining a certificate with certain Let's Encrypt authentication methods.

This article will explain the following:

- Authentication methods available in Certbot (HTTP-01, DNS-01, TLS-ALPN-01)
- Comparison of free DDNS services, including No-IP, DuckDNS, Dynu, and dynv6
- The relationship between TXT records and DNS-01 authentication
- The difference between sub-subdomains and wildcard DNS
- How to obtain a wildcard certificate
- How to choose a DDNS service suitable for your home server

*Note: The service specifications are based on publicly available information as of October 2026.*

---

## 1. How Certbot and Let's Encrypt Work

Let's Encrypt is a certificate authority (CA) that issues free SSL/TLS certificates.

Certbot is an ACME client that communicates with ACME-compliant CAs like Let's Encrypt to automate the process of obtaining and renewing certificates.

When obtaining a certificate, you need to prove that you manage the target domain.

This verification process is called **ACME Challenge (domain authentication)**.

### Comparison of ACME Authentication Methods

| Authentication Method | Port Used | Authentication Method | Wildcard Certificate |
|---|---|---|---|
| HTTP-01 | TCP 80 | Publish an authentication file via HTTP | No |
| DNS-01 | DNS | Register a TXT record | Yes |
| TLS-ALPN-01 | TCP 443 | Present authentication information via a TLS connection | No |

In standard Certbot usage, HTTP-01 and DNS-01 are commonly used.

TLS-ALPN-01 is defined in ACME, but it is not natively supported in Certbot.

### HTTP-01 Authentication

With HTTP-01, the CA accesses the following URL to verify domain ownership:

```text
http://example.com/.well-known/acme-challenge/<TOKEN>
```

For example, if you use Certbot's standalone mode, you would execute the following command:

```bash
sudo certbot certonly --standalone \
  -d example.com
```

In this command, Certbot starts a temporary HTTP server.

If you want to use an existing web server's public directory, you can also use webroot mode:

```bash
sudo certbot certonly --webroot \
  -w /var/www/html \
  -d example.com
```

The characteristics of HTTP-01 are:

- No need to edit DNS TXT records
- Relatively easy to use with standard free DDNS services
- Requires external access to TCP port 80
- Does not support wildcard certificates

If you are using Nginx or Apache, you can also use their respective Certbot plugins.

### DNS-01 Authentication

With DNS-01, you prove domain ownership by registering a TXT record in the target domain's DNS.

For example, if you are obtaining a certificate for `example.com`, you would create a record like this:

```text
Name:
_acme-challenge.example.com

Type:
TXT

Value:
<TOKEN specified by ACME>
```

The command to perform manual authentication is as follows:

```bash
sudo certbot certonly --manual \
  --preferred-challenges dns \
  -d example.com
```

DNS-01 has the following advantages:

- No need to open TCP port 80
- TCP port 443 is also not required during authentication
- Can obtain a certificate even if the web server is not publicly accessible
- Can obtain a wildcard certificate
- Can automate renewal with a DNS API

However, **to use DNS-01, you need a DNS service that allows you to register the necessary TXT records.**

Also, simply registering the TXT record manually does not allow for unattended automatic renewal. Automatic renewal requires a DNS API or an authentication script.

### TLS-ALPN-01 Authentication

TLS-ALPN-01 uses TCP port 443 and authenticates the domain using TLS ALPN extension and a dedicated temporary certificate.

Unlike HTTP-01, it does not use TCP port 80.

However, Certbot does not natively support this method, so if you need it, consider using another ACME client such as acme.sh or lego.

---

## 2. How to Run Certbot

Certbot allows you to choose the authentication method and the plugin or method to use, in addition to the authentication method.

| Option | Authentication Method | Use |
|---|---|---|
| `--nginx` | HTTP-01 | Integrates with Nginx |
| `--apache` | HTTP-01 | Integrates with Apache |
| `--webroot` | HTTP-01 | Uses an existing web server |
| `--standalone` | HTTP-01 | Starts a temporary HTTP server |
| `--manual --preferred-challenges http` | HTTP-01 | Manually executes HTTP authentication |
| `--manual --preferred-challenges dns` | DNS-01 | Manually registers TXT records |
| `--dns-cloudflare` | DNS-01 | Uses the Cloudflare API |
| `--dns-route53` | DNS-01 | Uses AWS Route 53 |
| `--dns-google` | DNS-01 | Uses Google Cloud DNS |
| `--dns-rfc2136` | DNS-01 | Uses RFC2136 for DNS updates |

DNS plugins may need to be installed separately from the Certbot main package.

## 3. Comparison of Free DDNS Services

On a home server, your global IP address may change.

Therefore, a DDNS (Dynamic DNS) is useful because it allows you to access the server using the same hostname even if the IP address changes.

Here, we will compare the following services:

- No-IP
- DuckDNS
- Dynu
- dynv6
- FreeDNS
- deSEC

The items to be compared are: TXT records, sub-subdomains, wildcard DNS, and support for Certbot DNS-01 authentication in the free plan.

### Free DDNS Comparison Table

| Service | TXT Record | Sub-Subdomain | Wildcard DNS | Certbot DNS-01 |
|---|---|---|---|---|
| No-IP | No | No | No | No |
| DuckDNS | Yes (limited) | Yes (for the same IP) | Limited (proprietary) | Yes (with external plugin) |
| Dynu | Yes | Yes | Yes | Yes |
| dynv6 | Yes | Yes | Yes | Yes |
| FreeDNS | Yes | Limited (depends on the domain) | No (free version) | Yes (manual authentication, etc.) |
| deSEC | Yes | Yes (within existing zone) | Yes | Yes (*New DDNS registration is currently suspended*) |

*Note: ○ indicates that it is available, △ indicates that it is available with conditions, and × indicates that it is not supported.
*Note: The table is for free DDNS hostnames provided by each service. Conditions may change if you use your own domain.
*Note: In FreeDNS's free plan, wildcard DNS is not supported in the official price list.

### No-IP

Website: [https://www.noip.com/](https://www.noip.com/)

No-IP is a well-known DDNS service.

For example, you can use a hostname like this:

```text
myserver.ddns.net
```

However, there are limitations in the free plan:

- Cannot use TXT records
- Cannot create 4th-level subdomains
- Cannot use wildcard DNS with free hostnames
- Cannot be used for standard DNS-01 authentication

For example, it does not support the following configuration:

```text
myserver.ddns.net
app.myserver.ddns.net
nas.myserver.ddns.net
```

However, HTTP-01 authentication is possible.

```bash
sudo certbot certonly --standalone \
  -d myserver.ddns.net
```

If you have an environment where you can open TCP port 80 and use a single hostname, it is an option.

Even with No-IP's paid Enhanced plan, there are limitations on 4th-level subdomains, and a custom domain and a corresponding DNS plan may be required.

Reference: [No-IP — Free and Enhanced Hostname Limitations](https://www.noip.com/support/knowledgebase/free-enhanced-limitations)

### DuckDNS

Website: [https://www.duckdns.org/](https://www.duckdns.org/)

With DuckDNS, you can obtain a free hostname like this:

```text
myserver.duckdns.org
```

DuckDNS provides a TXT record update API that is necessary for DNS-01.

Unlike standard DNS services, it has a special mechanism where TXT records are shared for sub-subdomains under the same DuckDNS hostname.

For example:

```text
myserver.duckdns.org
app.myserver.duckdns.org
nas.myserver.duckdns.org
```

These names use the shared TXT record mechanism of DuckDNS for TXT queries.

You can use this mechanism to perform DNS-01 authentication.

However, because TXT records cannot be managed independently for each hostname, there are limitations when performing multiple authentications simultaneously.

Reference: [DuckDNS API Specification](https://www.duckdns.org/spec.jsp)

### Dynu

Website: [https://www.dynu.com/](https://www.dynu.com/)

Dynu is a DDNS service with rich DNS features available for free.

The main features are:

- TXT records can be used
- Sub-subdomains can be created
- Wildcard DNS can be used
- Supports DNS API
- Supports RFC2136 for DNS updates

For example, you can create the following configuration:

```text
myserver.dynu.com
├── app.myserver.dynu.com
├── nas.myserver.dynu.com
└── git.myserver.dynu.com
```

With wildcard DNS, you can resolve all of these to the same IP address:

```text
*.myserver.dynu.com → 203.0.113.10
```

Using RFC2136, you can automate the process of updating TXT records with Certbot.

Reference: [Dynu DNS Features](https://www.dynu.com/DynamicDNS/Features)

### dynv6

Website: [https://dynv6.com/](https://dynv6.com/)

dynv6 is a free DDNS service that supports both IPv4 and IPv6.

You can use hostnames like this:

```text
myserver.dynv6.net
```

The main features are:

- Supports A, AAAA, TXT, and other DNS records
- Supports wildcard DNS
- Can set sub-subdomains
- Supports REST API
- Supports RFC2136 for DNS updates
- Suitable for IPv6 environments

For example, you can create the following configuration:

```text
myserver.dynv6.net
├── app.myserver.dynv6.net
├── nas.myserver.dynv6.net
└── git.myserver.dynv6.net
```

You can automate DNS record manipulation and combine it with Certbot's DNS-01 authentication.

Reference: [dynv6 API Documentation](https://dynv6.com/docs/apis)

### FreeDNS

Website: [https://freedns.afraid.org/](https://freedns.afraid.org/)

FreeDNS provides free DNS hosting and DDNS.

It supports many record formats such as TXT, A, AAAA, and CNAME.

However, when using a shared domain, there are restrictions on the names that can be used and the rights to edit records.

Also, **wildcard DNS is not available in the free plan.**

If you can set the necessary TXT records, you can use DNS-01, but unattended automatic renewal requires a separate renewal method.

Reference: [FreeDNS Membership Levels](https://freedns.afraid.org/pricing/)

### deSEC

Website: [https://desec.io/](https://desec.io/)

deSEC is a free DNS hosting service that supports DNSSEC and API-based DNS operations.

It has good compatibility with DNS-01 and can be used for automatic renewal of wildcard certificates.

However, as of October 2026, registration of new free DDNS hostnames (`dedyn.io`) is suspended.

Therefore, it is not a good option if you want to obtain a new free DDNS hostname.

However, it can still be used as a DNS management service for your own domain.

---

## 4. What is a Sub-Subdomain?

For example, suppose you have the following domain:

```text
example.com
```

A name with `app` added to it is a subdomain:

```text
app.example.com
```

Adding `test` to that makes it a sub-subdomain:

```text
test.app.example.com
```

The concept is the same for DDNS services:

```text
myserver.dynu.com
app.myserver.dynu.com
nas.myserver.dynu.com
```

`app` and `nas` are subdomains of `myserver.dynu.com`.

In terms of DNS hierarchy, `app.myserver.dynu.com` is a 4th-level name.

### It's Okay to Use the Same IP Address

When using multiple sub-subdomains, you do not need to assign a different IP address to each.

For example, it is okay to use the same IP address for all of them:

```text
app.myserver.dynu.com → 203.0.113.10
nas.myserver.dynu.com → 203.0.113.10
git.myserver.dynu.com → 203.0.113.10
```

In this case, you can use a reverse proxy such as Nginx to route to different services based on the hostname.

For example:

```text
app.myserver.dynu.com → localhost:3000
nas.myserver.dynu.com → localhost:5000
git.myserver.dynu.com → localhost:8080
```

DNS is responsible for resolving the hostname to the IP address, and the web server or reverse proxy is responsible for routing to the appropriate service.

---

## 5. What is Wildcard DNS?

Wildcard DNS is a mechanism to resolve multiple subdomains to the same IP address.

With a normal DNS configuration, you create a record for each hostname.

```text
app.myserver.dynu.com A 203.0.113.10
nas.myserver.dynu.com A 203.0.113.10
git.myserver.dynu.com A 203.0.113.10
```

With wildcard DNS, you can configure it like this:

```text
*.myserver.dynu.com A 203.0.113.10
```

This resolves names that do not have explicit records to the same IP address:

```text
app.myserver.dynu.com
nas.myserver.dynu.com
git.myserver.dynu.com
newapp.myserver.dynu.com
```

This eliminates the need to create DNS records for each new subdomain.

### Differences from Individual Records

| Item | Individual Sub-Subdomain | Wildcard DNS |
|---|---|---|
| Example | `app.myserver.dynu.com` | `*.myserver.dynu.com` |
| Adding Hostnames | Configure individually | Not required in principle |
| Combining into the same IP | ○ | ○ |
| Specifying different IPs | ○ | Use individual records |
| Managing many services | Increases the number of configurations | Easier to manage |

Note that DNS wildcards do not take precedence over existing explicit DNS records. Also, whether a wildcard matches or not depends on the DNS hierarchy and the state of existing records.

---

## 6. Difference Between Wildcard DNS and Wildcard Certificates

This is a part that is often confused.

**Wildcard DNS and wildcard certificates are different functions.**

Wildcard DNS is a mechanism for resolving hostnames to IP addresses.

A wildcard certificate is a TLS certificate that allows you to use HTTPS with multiple hostnames.

| Item | Wildcard DNS | Wildcard Certificate |
|---|---|---|
| Purpose | DNS name resolution | HTTPS authentication and encryption |
| Example | `*.example.com A 203.0.113.10` | `*.example.com` |
| Support for new hostnames | No need to add records in principle | No need to reissue certificates in principle |
| Certbot required | × | Can be used for obtaining |
| DNS-01 required | × | Required in Let's Encrypt |

### Scope of Wildcard Certificates

For example, if you obtain the following certificate:

```text
*.myserver.dynu.com
```

The hostnames that can be used are as follows:

| Hostname | Certificate Target |
|---|---|
| `app.myserver.dynu.com` | ○ |
| `nas.myserver.dynu.com` | ○ |
| `git.myserver.dynu.com` | ○ |
| `myserver.dynu.com` | × |
| `test.app.myserver.dynu.com` | × |

Wildcard certificates generally support one level of the hostname at the `*` position.

Therefore, if you want to use the root hostname with HTTPS, you need to specify both names in the certificate.

```bash
sudo certbot certonly \
  --manual \
  --preferred-challenges dns \
  -d myserver.dynu.com \
  -d '*.myserver.dynu.com'
```

With this command, you need to manually register a TXT record for each name for DNS-01 authentication.

Also, since two names require DNS-01 authentication, you may need to have multiple TXT values for the same `_acme-challenge` name. Check if your DNS service can handle multiple TXT values.

---

## 7. Using Certbot's DNS-01 Authentication with DuckDNS

DuckDNS provides a TXT record update API, so you can obtain a certificate without opening TCP port 80.

Certbot has a community-made plugin for DuckDNS.

### Install the Plugin

This is an example for Snap-installed Certbot.

```bash
sudo snap install certbot-dns-duckdns

sudo snap connect certbot:plugin certbot-dns-duckdns
```

Check if the plugin is recognized:

```bash
sudo certbot plugins
```

### Set the API Token

Find the API token on the DuckDNS management screen.

Create an authentication information file:

```text
/etc/letsencrypt/duckdns.ini
```

The content is as follows:

```ini
dns_duckdns_token = YOUR_DUCKDNS_TOKEN
```

Restrict access permissions:

```bash
sudo chmod 600 /etc/letsencrypt/duckdns.ini
```

### Obtain the Certificate

```bash
sudo certbot certonly \
  --authenticator dns-duckdns \
  --dns-duckdns-credentials /etc/letsencrypt/duckdns.ini \
  --dns-duckdns-propagation-seconds 60 \
  -d myserver.duckdns.org
```

To obtain a wildcard certificate, specify the target name as follows:

```bash
sudo certbot certonly \
  --authenticator dns-duckdns \
  --dns-duckdns-credentials /etc/letsencrypt/duckdns.ini \
  --dns-duckdns-propagation-seconds 60 \
  -d '*.myserver.duckdns.org'
```

DuckDNS has a special mechanism where TXT records are shared for sub-subdomains under the same DuckDNS hostname, so be careful when configuring multiple DNS-01 challenges simultaneously.

Reference: [certbot-dns-duckdns (GitHub)](https://github.com/infinityofspace/certbot_dns_duckdns)

---

## 8. Automating Certbot with Dynu and dynv6

Dynu and dynv6 support RFC2136 for DNS updates.

RFC2136 is a mechanism for dynamically changing DNS records.

By combining it with Certbot's `dns-rfc2136` plugin, you can automatically create and delete the TXT records required for DNS-01 authentication.

The basic steps are as follows:

1. Obtain a hostname from the DDNS service.
2. Configure a TSIG key for DNS updates.
3. Install Certbot's RFC2136 plugin.
4. Set the DNS server and authentication information in Certbot.
5. Obtain a certificate using DNS-01 authentication.
6. Test the renewal with `certbot renew --dry-run`.

The actual DNS server name, TSIG algorithm, and key format vary depending on the service.

Dynu's official documentation includes an example of creating a `_acme-challenge` TXT record using `nsupdate`.

dynv6 also supports RFC2136 and REST API.

Reference:

- [Dynu NSUpdate](https://www.dynu.com/DynamicDNS/IPUpdateClient/NSUpdate)
- [dynv6 API Documentation](https://dynv6.com/docs/apis)
- [Certbot Official User Guide](https://eff-certbot.readthedocs.io/en/stable/using.html)

---

## 9. Which Configuration is Recommended for Your Home Server?

The best configuration depends on your use case.

| Use Case | Recommended |
|---|---|
| TCP port 80 can be opened | HTTP-01 |
| TCP port 80 cannot be opened | DNS-01 |
| Single hostname is sufficient | No-IP, DuckDNS, etc. |
| Want to use TXT records | DuckDNS, Dynu, dynv6 |
| Want to use multiple sub-subdomains | Dynu, dynv6 |
| Want to use wildcard DNS | Dynu, dynv6 |
| Want to obtain a wildcard certificate | DNS-01-compatible DDNS |
| Want to automate Certbot | DNS API or RFC2136-compatible service |
| Want to use IPv6 as well | dynv6, etc. |

### Recommended Configuration Examples

For example, if you want to use dynv6 to host multiple web services on your home server, the following configuration is possible:

```text
DDNS:
myserver.dynv6.net

DNS:
*.myserver.dynv6.net
        ↓
  203.0.113.10
        ↓
     Nginx
        ├── app  → localhost:3000
        ├── nas  → localhost:5000
        └── git  → localhost:8080
```

The TLS certificate should target the following two names:

```text
myserver.dynv6.net
*.myserver.dynv6.net
```

This configuration has the following advantages:

- No need to change DNS settings even if you add new sub-subdomains
- Can run multiple web services on the same IP address
- Can share a wildcard certificate
- Can automate certificate renewal with DNS API or RFC2136

Note that DNS-01 is a method for obtaining certificates without opening ports, and it is not a mechanism for making the web service accessible from the outside. Separate measures such as a VPN or reverse tunnel are required for external access via HTTPS.

Also, note that if you are using CGNAT on your home network, you cannot achieve direct external access with DDNS alone.

---

## 10. Summary

When obtaining a free SSL/TLS certificate with Certbot, you usually use HTTP-01 or DNS-01.

HTTP-01 is relatively easy to set up, but requires external access to TCP port 80.

On the other hand, DNS-01 allows you to obtain a certificate without opening ports and supports wildcard certificates.

However, to use DNS-01, you need a DNS service that allows you to register the necessary TXT records.

No-IP's free plan can be used with HTTP-01, but it is not suitable for use cases that require TXT records or sub-subdomains.

DuckDNS allows you to easily use DNS-01, but there are limitations in managing TXT records.

Dynu and dynv6 support TXT records, sub-subdomains, wildcard DNS, and DNS APIs, making them suitable for home servers that run multiple services.

**If you want to use a free DDNS, have multiple sub-subdomains, and automate wildcard certificate renewal with Certbot, Dynu or dynv6 is a good choice.**

### References

- [Certbot Official Documentation](https://eff-certbot.readthedocs.io/en/stable/using.html)
- [Let's Encrypt Challenge Types](https://letsencrypt.org/docs/challenge-types/)
- [No-IP — Free and Enhanced Hostname Limitations](https://www.noip.com/support/knowledgebase/free-enhanced-limitations)
- [DuckDNS API Specification](https://www.duckdns.org/spec.jsp)
- [Dynu Dynamic DNS](https://www.dynu.com/)
- [dynv6](https://dynv6.com/)
- [FreeDNS](https://freedns.afraid.org/)
- [deSEC](https://desec.io/)
- [Certbot DuckDNS Plugin](https://github.com/infinityofspace/certbot_dns_duckdns)

---

*Last verified: October 20, 2026. The features, usage restrictions, and availability of new registrations for free services may change. Please check the official information of each service before implementation.*
