up:: [[Traefik]]
tags:: #traefik #on/traefik

# Configuration Discovery

![[/Extras/Images/providers.png]]

Configuration discovery in Traefik is achieved through _Providers_.

The _providers_ are infrastructure components, whether orchestrators, container engines, cloud providers, or key-value stores. The idea is that Traefik queries the provider APIs in order to find relevant information about routing, and when Traefik detects a change, it dynamically updates the routes.

---
## Docker
![[traefik-docker.png]]

[[Docker]] can use *labels* to allow Traefik to automatically configure the relevant [[routers]] and [[services]].
*Example*
```yaml
# Enabling the Docker provider
providers:
  docker: {}

# Attaching labels to containers (`compose.yaml`)
services:
  my-container:
    # container parameters
    labels:
      - traefik.http.routers.my-container.rule=Host(`example.com`)
```

By default Traefik watches for container level labels and can retrieve the private IP and port of those containers via the [[Docker]] API. If a container _exposes_ a single port, then Traefik uses this port for communication. If a container _exposes_ multiple ports; or no ports, then the port must be manually specified by using the label `trafik.http.servives.my-service.loadbalancer.server.port`

### Provider Configuration
```yaml
providers:
  # Enable Docker configuration backend
  docker:
    watch: true
    network: proxy_net
    swarmModeRefreshSeconds: "15"
    # Docker server endpoint. Can be a tcp or a unix socket endpoint.
    # Required
    # Default: "unix:///var/run/docker.sock"
    #
    endpoint: tcp://dockersocket:2375
    # Default host rule.
    #
    # Optional
    # Default: "Host(`{{ normalize .Name }}`)"
    #
    defaultRule: Host(`{{ index .Labels "com.docker.compose.service" }}.example.com`)
    # Expose containers by default in traefik
    #
    # Optional
    # Default: true
    #
    exposedByDefault: false
```

| Parameter               | Description                                                                            |
| ----------------------- | -------------------------------------------------------------------------------------- |
| watch                   | Traefik will monitor the Docker provider to events                                     |
| network                 | Defines a default docker network to use for connections to all containers              |
| swarmModeRefreshSeconds | Defines the refresh interval                                                           |
| endpoint                | Defines the [[Docker]] API endpoint (either local docker-socket or socket proxy)       |
| defaultRule             | Defines the routing rule applied to a container if no rule is specified by a [[label]] |
| exposedByDefault        | Expose containers by default. If set to false, containers that do not have a `traefik.enable=true` label are ignored.                                                                                       |

## File Provider
The file provider allows for _dynamic configuration_ in either a `YAML` or `TOML` file. It supports providing configuration through a *single configuration file* or *multiple separate files.*

*Example*
```yaml
# Enabling the file provider
providers:
  file:
    directory: "/path/to/file.yml"
```

*Declaring routers, middlewares and services example*
```yaml
http:
  # Add the router
  routers:
    router0:
      entryPoints:
        - web
      middlewares:
        - my-auth-middleware
      service: my-service
      rule: Path(`/path`)
  # Add the middleware
  middlewares:
    my-auth-middleware:
      basicAuth:
        users:
          - test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/
          - test2:$apr1$d9hr9HBB$4HxwgUir3HP4EsggP/QNo0
        usersFile: etc/traefik/.htpasswd
  # Add the service
  services:
    my-service:
      loadBalancer:
        servers:
          - url: http://foo/
          - url: http://bar/
        passHostHeader: false
```

### Provider Configuration
```yaml
providers:
  file:
    # For single file
    filename: /path/to/config/dynamic_conf.yml

    # For multiple files
    directory: /path/to/configs
    watch: true
```

It is recommended to use the `directory` method a-la [[nginx]] `conf.d` or `sites-available` directories. This then makes enabling, or disabling, routers, services and middlewares a snap.