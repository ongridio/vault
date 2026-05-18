---
title: systemd-resolved
source: https://wiki.archlinux.org/title/Systemd-resolved
kind: external
domain: dns
author: ArchWiki
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, dns]
---

> [!info] External article · imported reference
> Source: [wiki.archlinux.org](https://wiki.archlinux.org/title/Systemd-resolved)
> Author: ArchWiki
> License: cc-by-sa
> Fetched: 2026-05-18

# systemd-resolved

*systemd-resolved* is a [systemd](/title/Systemd "Systemd") service that provides network name resolution to local applications via a [D-Bus](/title/D-Bus "D-Bus") interface, the `resolve` [NSS](/title/Name_Service_Switch "Name Service Switch") service ([nss-resolve(8)](https://man.archlinux.org/man/nss-resolve.8)), and a local DNS stub listener on `127.0.0.53`. See [systemd-resolved(8)](https://man.archlinux.org/man/systemd-resolved.8) for the usage.

## Installation

*systemd-resolved* is a part of the [systemd](https://archlinux.org/packages/?name=systemd) package that is installed by default.

## Configuration

*systemd-resolved* provides resolver services for [Domain Name System (DNS)](https://en.wikipedia.org/wiki/Domain_Name_System "wikipedia:Domain Name System") (including [DNSSEC](https://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions "wikipedia:Domain Name System Security Extensions") and [DNS over TLS](https://en.wikipedia.org/wiki/DNS_over_TLS "wikipedia:DNS over TLS")), [Multicast DNS (mDNS)](https://en.wikipedia.org/wiki/Multicast_DNS "wikipedia:Multicast DNS") and [Link-Local Multicast Name Resolution (LLMNR)](https://en.wikipedia.org/wiki/Link-Local_Multicast_Name_Resolution "wikipedia:Link-Local Multicast Name Resolution").

The resolver can be configured by editing `/etc/systemd/resolved.conf` and/or drop-in *.conf* files in `/etc/systemd/resolved.conf.d/`. See [resolved.conf(5)](https://man.archlinux.org/man/resolved.conf.5).

To use *systemd-resolved* [start](/title/Start "Start") and [enable](/title/Enable "Enable") `systemd-resolved.service`.

**Tip**

To understand the context around the choices and switches, one can turn on detailed debug information for

*systemd-resolved*

as described in

[systemd#Diagnosing a service](/title/Systemd#Diagnosing_a_service "Systemd")

.

### DNS

Software that relies on glibc's [getaddrinfo(3)](https://man.archlinux.org/man/getaddrinfo.3) (or similar) will work out of the box, since, by default, `/etc/nsswitch.conf` is configured to use [nss-resolve(8)](https://man.archlinux.org/man/nss-resolve.8) if it is available.

To provide [domain name resolution](/title/Domain_name_resolution "Domain name resolution") for software that reads `/etc/resolv.conf` directly, such as [web browsers](/title/Web_browsers "Web browsers"), [Go](/title/Go "Go"), [GnuPG](/title/GnuPG "GnuPG") and [QEMU](/title/QEMU "QEMU") when using user networking, *systemd-resolved* has four different modes for handling the file—stub, static, uplink and foreign. They are described in [systemd-resolved(8) § /ETC/RESOLV.CONF](https://man.archlinux.org/man/systemd-resolved.8#/ETC/RESOLV.CONF). We will focus here only on the recommended mode, i.e. the stub mode which uses `/run/systemd/resolve/stub-resolv.conf`.

`/run/systemd/resolve/stub-resolv.conf` contains the local stub `127.0.0.53` as the only DNS server and a list of search domains. This is the recommended mode of operation that propagates the *systemd-resolved* managed configuration to all clients. To use it, replace `/etc/resolv.conf` with a symbolic link to it:

```
# ln -sf ../run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

#### Setting DNS servers

**Tip** To check the DNS currently in use by *systemd-resolved*, run `resolvectl status`.

##### Automatically

*systemd-resolved* will work out of the box with a [network manager](/title/Network_manager "Network manager") using `/etc/resolv.conf`. No particular configuration is required since *systemd-resolved* will be detected by following the `/etc/resolv.conf` symlink. This is going to be the case with [systemd-networkd](/title/Systemd-networkd "Systemd-networkd"), [NetworkManager](/title/NetworkManager "NetworkManager"), and [iwd](/title/Iwd "Iwd").

However, if the [DHCP](/title/DHCP "DHCP") and [VPN](/title/VPN "VPN") clients use the [resolvconf](https://en.wikipedia.org/wiki/resolvconf "wikipedia:resolvconf") program to set name servers and search domains (see [openresolv#Users](/title/Openresolv#Users "Openresolv") for a list of software that use *resolvconf*), the additional package [systemd-resolvconf](https://archlinux.org/packages/?name=systemd-resolvconf) is needed to provide the `/usr/bin/resolvconf` symlink.

##### Manually

In stub and static modes, custom DNS server(s) can be set in the [resolved.conf(5)](https://man.archlinux.org/man/resolved.conf.5) file:

```
/etc/systemd/resolved.conf.d/dns_servers.conf
```

```
[Resolve]
DNS=192.168.35.1 fd7b:d0bd:7a6e::1
Domains=~.
```

**Note**

- It is highly advised to use an [encrypted protocol](/title/Domain_name_resolution#Privacy_and_security "Domain name resolution") when connecting to third-party DNS services. See [#DNS over TLS](#DNS_over_TLS).
- Without the `Domains=~.` option in [resolved.conf(5)](https://man.archlinux.org/man/resolved.conf.5), *systemd-resolved* might use the per-link DNS servers, if any of them set `Domains=~.` in the per-link configuration.
- This option will not affect queries of domain names that match the more specific search domains specified in per-link configuration, they will still be resolved using their respective per-link DNS servers.

For more information on per-link configuration see [systemd.network(5)](https://man.archlinux.org/man/systemd.network.5).

##### Fallback

If *systemd-resolved* does not receive DNS server addresses from the [network manager](/title/Network_manager "Network manager") and no DNS servers are configured [manually](#Manually) then *systemd-resolved* falls back to the fallback DNS addresses to ensure that DNS resolution always works.

**Note**

The fallback DNS are in this order: Quad9, Cloudflare and Google. See the

[systemd PKGBUILD](https://gitlab.archlinux.org/archlinux/packaging/packages/systemd/-/blob/07bde3d1d8e6750a990d4f27825f9d48faac8676/PKGBUILD#L133-143)

where the servers are defined.

The addresses can be changed by setting `FallbackDNS` in [resolved.conf(5)](https://man.archlinux.org/man/resolved.conf.5). E.g.:

```
/etc/systemd/resolved.conf.d/fallback_dns.conf
```

```
[Resolve]
FallbackDNS=127.0.0.1 ::1
```

To disable the fallback DNS functionality set the `FallbackDNS` option without specifying any addresses:

```
/etc/systemd/resolved.conf.d/fallback_dns.conf
```

```
[Resolve]
FallbackDNS=
```

#### DNSSEC

**Warning**

DNSSEC support in systemd-resolved is considered experimental and incomplete as of Jun 2023.

[[1]](https://github.com/systemd/systemd/issues/25676#issuecomment-1634810897)

The [systemd](https://archlinux.org/packages/?name=systemd) package is built with DNSSEC validation disabled by default. This can be changed with the `DNSSEC` setting in [resolved.conf(5)](https://man.archlinux.org/man/resolved.conf.5).

- Set `DNSSEC=false` (the default) to disable DNSSEC validation.
- Set `DNSSEC=allow-downgrade` to validate DNSSEC only if the upstream DNS server supports it. If your DNS server does not support DNSSEC and you experience problems with this mode (e.g.: [systemd issue 21107](https://github.com/systemd/systemd/issues/21107), [systemd issue 36681](https://github.com/systemd/systemd/issues/36681)), you may need to explicitly disable DNSSEC validation instead.
- Set `DNSSEC=true` to always validate DNSSEC, thus breaking DNS resolution with name servers that do not support it. For example:

```
/etc/systemd/resolved.conf.d/dnssec.conf
```

```
[Resolve]
DNSSEC=true
```

Test DNSSEC validation by querying a domain name with no signature:

```
$ resolvectl query brokendnssec.net
```

```
brokendnssec.net: resolve call failed: DNSSEC validation failed: no-signature (RRSIG Missing)
```

Now test a domain with valid signature:

```
$ resolvectl query test.dnscheck.tools
```

```
test.dnscheck.tools: 2604:a880:400:d0::256e:b001 -- link: enp2s0
                   142.93.10.179               -- link: enp2s0

-- Information acquired via protocol DNS in 122.2ms.
-- Data is authenticated: yes; Data was acquired via local or encrypted transport: no
-- Data from: network
```

#### DNS over TLS

##### Global DNS over TLS

DNS over TLS is disabled by default. To enable it change the `DNSOverTLS` setting in the `[Resolve]` section in [resolved.conf(5)](https://man.archlinux.org/man/resolved.conf.5). To enable validation of your DNS provider's server certificate, include their hostname in the `DNS` setting in the format `ip_address#hostname`. For example:

```
/etc/systemd/resolved.conf.d/dns_over_tls.conf
```

```
[Resolve]
DNS=9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
DNSOverTLS=true
Domains=~.
```

**Note**

- This example uses Quad9. Replace it with a DNS resolver you trust. See [Domain name resolution#Third-party DNS services](/title/Domain_name_resolution#Third-party_DNS_services "Domain name resolution").
- With `DNSOverTLS=yes`, the DNS server used must support DNS over TLS. Otherwise all DNS requests will fail.
- Alternatively, it is possible to use DNS over TLS only if the server supports it with `DNSOverTLS=opportunistic`. If the used DNS server does not support DNS over TLS, systemd-resolved will fall back to regular unencrypted DNS.
- See the note in <#Manually> for specifics about using custom DNS servers globally.

[ngrep](https://archlinux.org/packages/?name=ngrep) can be used to test if DNS over TLS is working since DNS over TLS always uses port 853 and never port 53. The command `ngrep port 53` should produce no output when a hostname is resolved with DNS over TLS and `ngrep port 853` should produce encrypted output.

[Wireshark](/title/Wireshark "Wireshark") can be used for more detailed packet inspection of DNS over TLS queries.

##### Per-link DNS over TLS

Enabling DNS over TLS for specific connections depends on the [network manager](/title/Network_manager "Network manager"):

#### Additional listening interfaces

*systemd-resolved* answers DNS requests to local applications via loopback interface per default. To make *systemd-resolved* answer DNS requests on additional interfaces or addresses than the default one, set the option `DNSStubListenerExtra` for every additional interface in [resolved.conf(5)](https://man.archlinux.org/man/resolved.conf.5). For example:

```
/etc/systemd/resolved.conf.d/additional-listening-interfaces.conf
```

```
[Resolve]
DNSStubListenerExtra=192.168.10.10
DNSStubListenerExtra=2001:db8:0:f102::10
DNSStubListenerExtra=192.168.10.11:9953
```

**Tip**

This is useful, when using

*systemd-resolved*

on a

[router](/title/Router "Router")

which acts as a DNS server.

### mDNS

*systemd-resolved* is capable of working as a [multicast DNS](https://en.wikipedia.org/wiki/Multicast_DNS "wikipedia:Multicast DNS") resolver and responder.

The resolver provides [hostname](/title/Hostname "Hostname") resolution using a "*hostname*.local" naming scheme.

mDNS will only be activated for a connection if both systemd-resolved's mDNS support is enabled, and if the configuration for the currently active [network manager](/title/Network_manager "Network manager") enables mDNS for the connection.

*systemd-resolved'*s mDNS support is enabled by default. It can be disabled by its `MulticastDNS` setting (see [resolved.conf(5) § OPTIONS](https://man.archlinux.org/man/resolved.conf.5#OPTIONS)).

Enabling per-connection mDNS support depends on the [network manager](/title/Network_manager "Network manager"):

- For [systemd-networkd](/title/Systemd-networkd "Systemd-networkd"), set the `MulticastDNS` setting in the `[Network]` section of a per-connection settings file. You may also have to set `Multicast=true` in the `[Link]` section. See [systemd.network(5)](https://man.archlinux.org/man/systemd.network.5).
- For [NetworkManager](/title/NetworkManager "NetworkManager"), set `mdns` in the `[connection]` section of the connection's settings file. Running `nmcli connection modify interface_name connection.mdns {yes|no|resolve}` will do that for you. See [nm-settings(5)](https://man.archlinux.org/man/nm-settings.5).

**Note**

- If [Avahi](/title/Avahi "Avahi") has been installed, consider [disabling](/title/Disabling "Disabling") or [masking](/title/Mask "Mask") `avahi-daemon.service` and `avahi-daemon.socket` to prevent conflicts with *systemd-resolved*.
- If you plan to use mDNS and a [firewall](/title/Firewall "Firewall"), make sure to open UDP port `5353`.
- If you plan to use [Avahi](/title/Avahi "Avahi") to provide DNS-SD/mDNS printer discovery in [cups#Printer_discovery](/title/Cups#Printer_discovery "Cups") but still want to have *systemd-resolved* to resolve and cache mDNS requests, `MulticastDNS=resolve` in `resolved.conf`. Thus, whenever *systemd-resolved* and *Avahi* are running at the same time, *Avahi* will respond to mDNS requests but *systemd-resolved* will cache them on the network.

### LLMNR

[Link-Local Multicast Name Resolution](https://en.wikipedia.org/wiki/Link-Local_Multicast_Name_Resolution "wikipedia:Link-Local Multicast Name Resolution") is a [hostname](/title/Hostname "Hostname") resolution protocol created by Microsoft.

LLMNR will only be activated for the connection if both the systemd-resolved's global setting (`LLMNR` in [resolved.conf(5) § OPTIONS](https://man.archlinux.org/man/resolved.conf.5#OPTIONS)) and the [network manager's](/title/Network_manager "Network manager") per-connection setting is enabled. By default *systemd-resolved* enables LLMNR responder; [systemd-networkd](/title/Systemd-networkd "Systemd-networkd") and [NetworkManager](/title/NetworkManager "NetworkManager")[[3]](https://gitlab.freedesktop.org/NetworkManager/NetworkManager/issues/301) enable it for connections.

If you plan to use LLMNR and use a [firewall](/title/Firewall "Firewall"), make sure to open UDP and TCP ports `5355`.

## Lookup

To query DNS records, mDNS or LLMNR hosts you can use the *resolvectl* utility.

For example, to query a DNS record:

```
$ resolvectl query archlinux.org
```

```
archlinux.org: 2a01:4f8:172:1d86::1
               138.201.81.199

-- Information acquired via protocol DNS in 48.4ms.
-- Data is authenticated: no
```

## Troubleshooting

### systemd-resolved not searching the local domain

*systemd-resolved* may not search the local domain when given just the hostname, even when `UseDomains=true` or `Domains=[domain-list]` is present in the appropriate [systemd-networkd](/title/Systemd-networkd "Systemd-networkd")'s *.network* file, and that file produces the expected `search [domain-list]` in `resolv.conf`. You can run `networkctl status` or `resolvectl status` to check if the search domains are actually being picked up.

Possible workarounds:

- Disable [#Global DNS over TLS](#Global_DNS_over_TLS) on the local link by setting `DNSOverTLS=false` in the *.network* file of the link in question. It should still be using TLS on non-local DNS queries, even if they're leaving through the same interface.
- Disable [LLMNR](#LLMNR) to let *systemd-resolved* immediately continue with appending the DNS suffixes
- Trim `/etc/nsswitch.conf`'s `hosts` database (e.g., by removing `[!UNAVAIL=return]` option after `resolve` service)
- Switch to using fully-qualified domain names
- Use `/etc/hosts` to resolve hostnames
- Fall back to using glibc's `dns` instead of using systemd's `resolve`

### systemd-resolved does not resolve hostnames without suffix

To make systemd-resolved resolve hostnames that are not fully qualified domain names, add `ResolveUnicastSingleLabel=true` to `/etc/systemd/resolved.conf`.

**Warning**

This will forward single-label names to global DNS servers which may not be under your control. This behaviour is not standard-conformant and may create a privacy and security risk. See

[resolved.conf(5)](https://man.archlinux.org/man/resolved.conf.5)

for details.

This only seems to work with LLMNR disabled (`LLMNR=false`).

If you are using [systemd-networkd](/title/Systemd-networkd "Systemd-networkd"), you might want the domain supplied by the DHCP server or IPv6 Router Advertisement to be used as a search domain. This is disabled by default, to enable it add to the interface's *.network* file:

```
[Network]
UseDomains=true
```

You can check what systemd-resolved has for each interface with:

```
$ resolvectl domain
```

## See also
