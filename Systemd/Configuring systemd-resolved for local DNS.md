# Configuring Linux systemd-resolved for local DNS servers

## Introduction

The **systemd-resolved** service is a fundamental component of modern Linux systems that handles network name resolution, DNS caching, and LLMNR (Link-Local Multicast Name Resolution). As part of the [systemd ecosystem](https://www.freedesktop.org/wiki/Software/systemd/), it provides a unified approach to managing DNS queries and network name resolution across different network interfaces and configurations. However, it is that very unified approach that conflicts with maintaining a self-hosted DNS server such as [BIND9](https://www.isc.org/bind/) on the local host or via a container.

### The Problem

The systemd-resolved service, when operating in its default configuration, starts a DNS stub listener on the ports that any DNS server uses (TCP/UDP Port 53) that binds to 127.0.0.53. Previously, all DNS client operations were handled in the file ```/etc/resolv.conf``` and those configurations were easy to maintain (an example of a classic ```resolv.conf``` is below).

```conf
nameserver a.b.c.d
nameserver w.x.y.z
domain example.com
search example.com sub.example.com
```

This, rather simplistic configuration tells the system to use the DNS nameservers ```a.b.c.d``` and ```w.x.y.z``` for name resolution. If a request for a non-fully qualified host was presented (i.e. ```hostA```) the system resolver would search the listed domains for a match.

Systemd-resolved changes all that with its stub listener:

```conf
nameserver 127.0.0.53
options edns0 trust-ad
```

Which can also be checked by issuing the command ```resolvectl status``` to provide the following output:

```bash
Global
       Protocols: +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub
Current DNS Server: 8.8.8.8
       DNS Servers: 8.8.8.8 8.8.4.4

Link 2 (enp0s3)
    Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6 mDNS/IPv4 mDNS/IPv6
         Protocols: +DefaultRoute +LLMNR +mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 192.168.1.1
       DNS Servers: 192.168.1.1
        DNS Domain: localdomain
```

> [!NOTE]
> The output line ```resolv.conf mode: stub``` indicates that systemd-resolved is running the stub listener.

<details>

<summary> A second verification to check if the stub resolver is being used is to check the contents of /etc/resolv.conf which, in a default setup, would show the nameserver as 127.0.0.53:</summary>

```conf
user@lab:~$ cat /etc/resolv.conf
# This is /run/systemd/resolve/stub-resolv.conf managed by man:systemd-resolved(8).
# Do not edit.
#
# This file might be symlinked as /etc/resolv.conf. If you're looking at
# /etc/resolv.conf and seeing this text, you have followed the symlink.
#
# This is a dynamic resolv.conf file for connecting local clients to the
# internal DNS stub resolver of systemd-resolved. This file lists all
# configured search domains.
#
# Run "resolvectl status" to see details about the uplink DNS servers
# currently in use.
#
# Third party programs should typically not access this file directly, but only
# through the symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a
# different way, replace this symlink by a static file or a different symlink.
#
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

nameserver 127.0.0.53
options edns0 trust-ad
search .
```

</details>

Granted, systemd-resolved _does still forward requests_ to an upstream server - just as the "classic" resolv.conf did - and cache any responses it receives. The issue is that the stub is occupying the ports needed to run a normal DNS server.

### The Fix

Most guides document that a system admin should edit ```/etc/systemd/resolved.conf``` directly. While this **does work** it can (and often does) get overwritten by package updates which can end up breaking things. The _correct, upgrade safe_ method is to take advantage of "drop-in" configuration. The [man page](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html) states (emphasis mine):

> [!TIP]
> Local overrides can also be created by creating drop-ins, as described below. The main configuration file can also be edited for this purpose (or a copy in /etc/ if it is shipped under /usr/), however **using drop-ins for local configuration is recommended** over modifications to the main configuration file.

Whilst the main configuration file (```/etc/systemd/resolved.conf```) can be edited directly it is not recommended.

<details>
<summary> The default systemd-resolved configuration file.</summary>

```conf
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it under the
#  terms of the GNU Lesser General Public License as published by the Free
#  Software Foundation; either version 2.1 of the License, or (at your option)
#  any later version.
#
# Entries in this file show the compile time defaults. Local configuration
# should be created by either modifying this file (or a copy of it placed in
# /etc/ if the original file is shipped in /usr/), or by creating "drop-ins" in
# the /etc/systemd/resolved.conf.d/ directory. The latter is generally
# recommended. Defaults can be restored by simply deleting the main
# configuration file and all drop-ins located in /etc/.
#
# Use 'systemd-analyze cat-config systemd/resolved.conf' to display the full config.
#
# See resolved.conf(5) for details.

[Resolve]
# Some examples of DNS servers which may be used for DNS= and FallbackDNS=:
# Cloudflare: 1.1.1.1#cloudflare-dns.com 1.0.0.1#cloudflare-dns.com 2606:4700:4700::1111#cloudflare-dns.com 2606:4700:4700::1001#cloudflare-dns.com
# Google:     8.8.8.8#dns.google 8.8.4.4#dns.google 2001:4860:4860::8888#dns.google 2001:4860:4860::8844#dns.google
# Quad9:      9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
#
# Using DNS= configures global DNS servers and does not suppress link-specific
# configuration. Parallel requests will be sent to per-link DNS servers
# configured automatically by systemd-networkd.service(8), NetworkManager(8), or
# similar management services, or configured manually via resolvectl(1). See
# resolved.conf(5) and systemd-resolved(8) for more details.
#DNS=
#FallbackDNS=
#Domains=
#DNSSEC=no
#DNSOverTLS=no
#MulticastDNS=no
#LLMNR=no
#Cache=yes
#CacheFromLocalhost=no
#DNSStubListener=yes
#DNSStubListenerExtra=
#ReadEtcHosts=yes
#ResolveUnicastSingleLabel=no
#StaleRetentionSec=0
#RefuseRecordTypes=
```

</details>

Indeed, best practice would have it that any parameter that is to be changed from the default (i.e. DNSStubListener=yes to DNSStubListener=no) should be placed in a drop-in file. Drop-in files are easy enough to create and maintain as they all live in the same place for the particular systemd service being configured (in this case, systemd-resolved).

#### Step 1

By default, there are no override / drop-in directories for any systemd service (unless an installed package creates one) so the very first step is to create the required directory:

```bash
user@host:~$ sudo mkdir -p /etc/systemd/resolved.conf.d/
```

> Note: strictly speaking, the ```-p``` (make parents as needed) switch in the ```mkdir``` command isn't needed as the directory ```/etc/systemd``` should already exist in a system managed by systemd.

#### Step 2

Once the directory has been created, it is time to create the drop-in configuration file(s). In most cases when wanting to run a DNS server, all that is needed is to disable the stub listener which can be achieved by the following:

- Create the file

```bash
user@host:~$ sudo nano /etc/systemd/resolved.conf.d/60-stub.conf
```

> Note: It is recommended that any override files be prefixed with a two-digit number. For user created overrides, the [man page](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html) recommends the prefix of 60 through to 90 (i.e. 60-stub.conf, 70-dns.conf, etc.) to allow packages to place _their overrides_ at a lower priority to avoid package changes disrupting user defined configurations.

#### Step 3

- Add the relevant parameter to override/change

```conf
[Resolve]
DNSStubListener=no
```

- Save the configuration file using the _control (CTRL)_ key and O (to write the file) then CTRL+X to exit
  - Alternatively, save and exit by using CTRL+X

#### Step 4

- restart the systemd-resolved service:

```bash
user@host:~$ sudo systemctl restart systemd-resolved
```

#### Step 5

Confirm that systemd-resolved is no longer running the stub listener by issuing the command ```resolvectl status``` which should display output similar to this:

```bash
Global
         Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
  resolv.conf mode: uplink
       DNS Servers: 1.1.1.1 1.0.0.1

Link 2 (eth0)
    Current Scopes: DNS
         Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
       DNS Servers: 1.1.1.1 1.0.0.1
     Default Route: yes
```

> Note that the ```resolv.conf mode``` is now showing as **uplink** instead of _stub_.

Now systemd-resolved is no longer occupying port 53 so a DNS server can now be installed and started without issue.
