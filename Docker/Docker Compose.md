# Docker-Compose

The Compose file is a [YAML](https://yaml.org/) file defining services, networks, and volumes for a Docker application. The latest and recommended version of the Compose file format is defined by the [Compose Specification](https://github.com/compose-spec/compose-spec/blob/master/spec.md). The Compose spec merges the legacy 2.x and 3.x versions, aggregating properties across these formats and is implemented by **Compose 1.27.0+**.

## Top Level Elements

The compose file is a YAML file defining [[#services]], [[#networks]], [[#volumes]], [[#configs]] and [[#secrets]]. The default name for a compose file is `compose.yaml` (preferred) or `compose.yml` in the current working directory. Compose file implementations also support `docker-compose.yaml` and `docker-compose.yml` for backwards compatibility. If both files exist, `compose.yaml` is used.

> The *version* TLE has been deprecated and is no longer required.
> Additionally, the *name* TLE is used to set a project, or stack, name.

### services

The *services* top-level element (TLE) of the *docker-compose* file is the meat & potatoes of the file. It defines the application or service that the container is to run. Each docker-compose.yml file **must** include a services TLE.

*Example:*

```yaml
services:
  my-app:
    container_name: my-app
    image: myapp/image:latest
    ports:
      - 80:80
    restart: unless-stopped
    volumes:
      - my-app-volume:/etc/my-app-config
```

### networks

By default Docker-Compose will create a new network for the given compose file. You can change the behavior by defining custom networks in your compose file.

#### Create and assign custom network

*Example:*

```yaml
networks:
  custom-network:
    name: my-custom-network
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
        - ip_range: 172.28.5.0/24
        - gateway: 172.28.5.254

services:
  app:
    networks:
      - custom-network
```

#### Use existing networks

If you want to use an existing Docker network for your compose files, you can add the `external: true` parameter in your compose file
*Example:*

```yaml
networks:
  existing-network:
    external: true
```

### volumes

Volumes are persistent data stores implemented by the platform. The Compose specification offers a neutral abstraction for services to mount volumes, and configuration parameters to allocate them on infrastructure.

The `volumes` section allows the configuration of named volumes that can be reused across multiple services. Here’s an example of a two-service setup where a database’s data directory is shared with another service as a volume named `db-data` so that it can be periodically backed up:

#### Example Volumes

```yaml
services:
  backend:
    image: awesome/database
    volumes:
      - db-data:/etc/data

  backup:
    image: backup-service
    volumes:
      - db-data:/var/lib/backup/data

volumes:
  db-data:
    name: "my-db-data"
```

An entry under the top-level `volumes` key can be empty, in which case it uses the platform's default configuration for creating a volume. Alternatively, if the volume exists, the `external: true` parameter can be added:

#### More Example Volumes

```yaml
volumes:
  db-data:
    external: true
      name: "actual-volume-name"
```

### configs

Configs allow services to adapt their behaviour without the need to rebuild a Docker image. Configs are comparable to Volumes from a service point of view as they are mounted into service’s containers filesystem. The actual implementation detail to get configuration provided by the platform can be set from the Configuration definition.

As with [[#networks]] and [[#volumes]], the *configs* key can also have the `external: true` parameter set.

#### Example Configs

```yaml
configs:
  http_config:
    file: ./httpd.conf

configs:
  http_config:
    external: true
```

### secrets

Secrets are a flavour of Configs focussing on sensitive data, with specific constraint for this usage. As the platform implementation may significantly differ from Configs, dedicated Secrets section allows to configure the related resources.

The top-level `secrets` declaration defines or references sensitive data that can be granted to the services in this application. The source of the secret is either `file` or `external`.

- `file` : The secret is created with the contents of the file at the specified path
- `environment` :  The secret is created with the value of an environment variable
- `external` : If set to `true`, specifies that this secret has already been created.
- `name` : The name of the secret object in [[Docker]]. This field can be used to reference secrets that contain special characters.

### Example Secrets

The `server-certificate` secret is created as `<project_name>_server-certificate` when the application is deployed.

```yaml
secrets:
  server-certificate:
    file: ./server.cert
```

## Fragments

It is possible to re-use configuration elements - particularly in long, multi-service, compose files by using YAML anchors.

### Example Fragments

```yaml
volumes:
  db-data: &default-volume
    driver: default
  metrics: *default-volume*
```

In previous sample, an *anchor* is created as `default-volume` based on `db-data` volume specification. It is later reused by *alias* `*default-volume` to define `metrics` volume.

It is also possible to partially override values set by anchor reference using the [YAML merge type](http://yaml.org/type/merge.html). In following example, `metrics` volume specification uses alias to avoid repetition but override `name` attribute:

```yaml
services:
  backend:
    image: awesome/database
    volumes:
      - db-data
      - metrics
volumes:
  db-data: &default-volume
    driver: default
    name: "data"
  metrics:
    <<: *default-volume
    name: "metrics"
```

## Variables & Interpolation

Values set in a compose file can be set by variables. Compose files use a Bash-like syntax `${VARIABLE}`.

Both `${VARIABLE}` and `$VARIABLE` syntax are supported. Default values can be defined inline using typical shell syntax.

- `${VARIABLE:-default}` evaluates to `default` if `VARIABLE` is unset or empty in the environment.
- `${VARIABLE-default}` evaluates to `default` only if `VARIABLE` is unset in the environment.

### Example

```yaml
secrets:
  my-super-secret:
    file: $SECRETSDIR/my-super-secret
services:
  backend:
    image: awesome/database
    volumes:
      - db-data:/etc/data
    env_file: .env
```

The variable `$SECRETSDIR` in the example above would refer to a pre-defined path. Often, variables are set in a separate `.env` file located alongside the `compose.yaml` file. The `env_file` parameter can be used to define a specific environment file, or list of files.

> **NOTE** variables present in multiple `.env` files are applied in a top-down fashion. As such, if a variable exists in the first file **and** one or more subsequent files, the **last** value set is used.

Each line in an `.env` file **must** be in `VAR[=[VAL]]` format. Comments are denoted by `#` and are to be ignored.

### Example Env File

```yaml
# Set Rails/Rack environment
RACK_ENV=development
VAR="quoted"
```

In the above example, the variable `RACK_ENV` is passed as-is and not modified. The value of `VAR` is also passed as-is *with the quotation marks intact*. This is often required for shell-variables and variables that must pass special characters.

Environment variables can also be defined in the `services` top-level element with the following rules:

- Boolean values (yes, no, true, false) to be enclosed in quotation marks
- Values reference a stored variable *must* be enclosed in single-quotes

```yaml
services:
  my-app:
    container_name: my-app
    image: myapp/image:latest
    ports:
      - 80:80
    restart: unless-stopped
    volumes:
      - my-app-volume:/etc/my-app-config
    environment:
      - RACK_ENV: development
      - SHOW: "true"
      - VAR: '$VAR'
```

Docker Compose Specification: [Compose specification | Docker Documentation](https://docs.docker.com/compose/compose-file/)
