up:: [[Traefik]]
tags:: #on/traefik 

# Routing & Load Balancing Overview
![[architecture-overview.png]]
When [[Traefik]] is started, [[Entrypoints]] are defined (these are port numbers to listen on). Then, connected to these entrypoints, [[routers]] analyse the incoming requests to see if they match a set of rules. If they do match, the router might transform the request using pieces of [[middlewares]] before forwarding them to the relevant [[services]].

Each aspect of the Traefik architecture defines clear responsibilities:
- [[Configuration Discovery|Providers]] discover the services that live on the infrastructure
- [[Entrypoints|Entrypoints]] listen for incoming traffic
- [[routers|Routers]] analyse the request (host, path, headers, SSL,...)
- [[services|Services]] forward the request to the relevant server or container (load balancing)
- [[middlewares|Middlewares]] may update the request or make additional decisions based on the request (authentication, rate limiting, header manipulation,...)