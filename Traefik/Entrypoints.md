up:: [[Routing & Load Balancing Overview]]
tags:: #on/traefik 
reference:: [Traefik EntryPoints Documentation - Traefik](https://doc.traefik.io/traefik/routing/entrypoints/)

# EntryPoints
![[entrypoints.png]]
EntryPoints are the network entry points into the Traefik process. They define the port which will receive the packets, and whether to listen for TCP or UDP.

## Configuration
EntryPoints are part of the *static configuration* and can be defined by using a file, or CLI arguments.
*Example - Port 80 & 443*
```yaml
entryPoints:
  http:
    address: ":80"
  https:
    address: ":443"
```
The above example defines two entry points. One called "http" and the the other called "https".

*Example - UDP on port 1704*
```yaml
entryPoints:
  streaming:
    address: ":1704/udp"
```
This example creates an entrypoint called "streaming" that will listen on UDP port 1704.

| Parameter       | Description                                                                                                                   |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| address         | The address defines the port, and optionally the hostname on which to listen. If no protocol is specified, the default is TCP |
| `host:3179` | Defines an entryPoint that will listen on TCP port 3179 on the name `host` only.                                              |

The parameter `forwardedHeaders` can be added to allow Traefik to trust the forwarded headers information (`X-Forwarded-*`). It is often used in conjunction with the `trustedIPs` sub-parameter:
```yaml
entryPoints:
  http:
    address: ":80"
    forwardedHeaders:
      trustedIPs:
        - "127.0.0.1/32"
        - "192.168.1.7"
```
This is often useful when behind a Content Delivery Network (CDN) such as CloudFlare so that the visitors *real IP address* can be (relatively) safely determined via the preserved headers.
