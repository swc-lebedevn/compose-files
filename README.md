# Docker compose file for PoolParty

This repository contains docker compose files for running PoolParty and related services.
Docker compose simplifies the deployment of several services in one go, their configuration, and the relation between
the service.

Docker compose allows chaining multiple service definition files, allowing the creation of multiple flavors, e.g.
development, stage, and production.

There are several files in this repository:
* `docker-compose.yaml` this file should be used only for evaluation and testing purposes - it's not sutable for production
* `production.yaml` this build on the default configuration, by adding additional services and configuration
* `spark.yaml` if desired, a separate spark instance can be deployed
* `ssl.yaml` add support for ssl to the proxy service

The basic docker compose commands are:

`docker compose up`: this is used to start all services defined in the compose files. It will start the services in the
foreground. To start the container and detach, add the `-d`/`--detach` flag.

`docker compose down`: this is used to stop all services defined in the compose files. To remove all volumes add the
`--volumes` flag, but be careful, as this will delete all data used by the services.

To control individual containers you can use `docker compose [COMMAND]`, where `[COMMAND]` can be `start`, `stop`,
`restart`, etc. Check the output of `docker compose help` for more information.

Multiple files could be used to manage different flavors of the deployment. The default `docker-compose.yaml` can be
used for development and testing/evaluations. All other flavors build on top of this, for example to deploy a separate
Apache Spark service use the following command:

```shell
docker compose -f docker-compose.yaml -f spark.yaml up
```

# Running

## Prerequisites

1. A recent version of [Docker](https://docs.docker.com/engine/install/) and the docker-compose plugin.
   1. Installation instructions for Docker on Linux include the `docker-compose-plugin`

## Configuration

The services are configured using environment variables. The repository contains a [.env_template](.env_template)
file containing most common configurations.

Before running any `docker compose` commands, copy the `.env_template` as `.env` in the same directory and change the 
configurations as desired.

There are two variables that must be changed:
* `POOLPARTY_LICENSE` this is the full path on the host machine to a valid PoolParty license
* `GRAPHDB_LICENSE` this is the full path on the host machine to a valid GraphDB license

Other notable variable are:
* `POOLPARTY_KEYCLOAK_ADMIN_USERNAME`: this is the admin username for Keycloak, default `poolparty_auth_admin`.
* `POOLPARTY_KEYCLOAK_ADMIN_PASSWORD`: this is the password for the admin user in Keycloak, default `admin` and it's 
recommended to be changed in production environments.
* `POOLPARTY_SUPER_ADMIN_PASSWORD`: this is the password for the `superadmin` in PoolParty, default `poolparty`. After the
first login you'll be asked to change this password.

Review the comments in the [.env_template](./.env_template) for all available variable and their purpose.

## Development

This should be used only for development, testing, and evaluation purposes. The services are configured with less 
security and no high-availability.

To start a local development environment, run the following command:

```shell
docker compose up -d
```

This command will use the default [docker-compose.yaml](./docker-compose.yaml) file. It will start PoolParty and the services that it
depends on.

After all services are running, PoolParty should be accessible at http://poolparty.127.0.0.1.nip.io/PoolParty. 
The default password for the `superadmin` is `poolparty`. After the first login, you will be prompted to change your 
password.

You can use a different instance for the Keycloak service. There are a few thing to configure before running without 
Keycloak by default:
1. Remember to change the PoolParty configurations for Keycloak.
2. Update the proxy configuration file
   1. Create a copy of either [proxy.conf](files/nginx/poolparty.conf) or [proxy_ssl.conf](files/nginx/proxy_ssl.conf) and remove the `/auth` location directive.
   2. In `.env` change the `PROXY_CONFIG_PATH` variable to point to your new config.
3. Use the following command to start without deploying Keycloak.
```shell
docker compose up -d --scale keycloak=0
```

## Production

The production deployment builds on the default compose file. To deploy the production configuration, run:

```shell
docker compose -f docker-compose.yaml -f production.yaml up -d
```

Here we use multiple compose files, where every file is merged with the previous one. By doing that we add additional
services or change configurations on existing ones.

The additional [production.yaml](./production.yaml) file configures additional service, e.g. PostgreSQL service, used
by Keycloak, and other optimization options.

## Spark

Starting PoolParty 10, support for external Spark service was added. If needed any of the above deployments can be
extended with an external Spark. To do that run a command similar to the following:

```shell
docker compose up -f docker-compose.yaml -f spark.yaml up -d
```

## Running with SSL

In order to run the `proxy` service with SSL enabled, you will need to:
1. Obtain an SSL certificate and key. Preferably, this should be generated by a trusted authority.
   1. To generate a self-signed certificate, you can use the following command:
   ```shell
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout /etc/ssl/private/poolparty.key \
      -out /etc/ssl/certs/poolparty.crt
   ```
2. In `.env` update the values of `PROXY_CERT_PATH` and `PROXY_CERT_KEY_PATH` to point to the respective files.
3. In `.env` update the value of `PROXY_CONFIG_PATH` to `./files/nginx/proxy_ssl.conf`
4. Finally, start the services with:
```shell
docker compose -f docker-compose.yaml -f ssl.yaml up -d
```

# Stopping services

If the services were started in the foreground, you can simply interrupt the process and the services will stop. If 
started with `-d` flag - the command is the same as the one for starting, but instead of `up`, specify the `down`
command.

The delete all data stored by the services, append the `--volumes` flag.

If you have specified multiple compose files in the `up` command, also specify them for the `down` command, otherwise
some services might be left running.

# Viewing log messages

To view the combined log messages of all services use the `docker compose logs` command. If you have specified multiple
compose files when starting the services, specify them here as well.

Instead of viewing the logs from all services, you can request the logs from a single service. Using
`docker compose logs [SERVICE]`, where [SERVICE] is the name of the desired service as defined in the compose file.
You can use `docker compose logs -f [SERVICE]`, to follow the log as it's updated.

# Migration

[Upgrading PoolParty 9.7 to 10](https://help.graphwise.ai/en/how-to-install---manage-graphwise-components/installation---migration/poolparty-9-7-to-10-migration-guide.html)
