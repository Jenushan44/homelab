# Docker Deployment

## Overview

Docker is used to run Nextcloud and PostgreSQL in separate containers on the Ubuntu Server VM. The deployment uses prebuilt Docker images instead of custom Dockerfiles because both Nextcloud and PostgreSQL already have images that contain the software needed to run them. Docker Compose is used to configure and manage both services together.

## Containers

The deployment uses two main containers:

- **Nextcloud container** - runs the Nextcloud web application.
- **PostgreSQL container** - runs the PostgreSQL database used by Nextcloud.

The containers are separate from each other but are connected through the Docker Compose network.

## Docker Images

The Nextcloud service uses:

```yaml
image: nextcloud:latest
```

This image already contains the Nextcloud application and the environment needed to run it.

The PostgreSQL service uses:

```yaml
image: postgres:16
```

This image contains PostgreSQL and is used to create the database container. Since both of the services use prebuilt Docker images, I did not need to create custom Dockerfiles.

## Docker Compose

Docker Compose manages both containers through one `compose.yaml` file.

The Compose file defines:

- Docker images
- environment variables
- persistent volumes
- port mappings
- restart policies
- communication between services

The services can be started using:

```bash
sudo docker compose up -d
```

The `up` command creates and starts the services defined in the Compose file. The `-d` option runs the containers in detached mode so they continue running in the background after the terminal is closed.

The services can be stopped and removed using:

```bash
sudo docker compose down
```

Running `docker compose down` removes the containers but does not remove the named volumes unless the volumes are included in the command.

## Environment Variables

Environment variables are used to give the containers configuration values without hardcoding everything directly into the application.

The PostgreSQL container receives:

```text
POSTGRES_DB
POSTGRES_USER
POSTGRES_PASSWORD
```

These values tell PostgreSQL which database and user to create.

Nextcloud receives the same database information along with:

```text
POSTGRES_HOST=postgres
```

The `POSTGRES_HOST` value tells Nextcloud where the PostgreSQL service is located. The sensitive values are stored in a `.env` file and referenced from `compose.yaml`.

For example:

```yaml
POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

This keeps the password separate from the main Compose configuration.

## Docker Networking

Docker Compose automatically creates a private network for the services in the project. Both the Nextcloud and PostgreSQL containers are connected to this network which means means Nextcloud can connect to PostgreSQL using the Compose service name instead of needing to know the PostgreSQL container's internal IP address:

```text
postgres
```

Database Connection:

```text
Nextcloud Container
        |
        | postgres:5432
        |
        |--->  PostgreSQL Container
```

PostgreSQL uses port `5432` inside the Docker network and the PostgreSQL port is not published to the Ubuntu Server VM because nothing outside of the Docker network needs to access the database directly.

## Nextcloud Port Mapping

The Nextcloud web server listens on port `80` inside the container.

Docker maps port `8081` on the Ubuntu Server VM to port `80` inside the Nextcloud container:

```yaml
ports:
  - "8081:80"
```

```text
Ubuntu VM :8081
      |
      | Docker port mapping
      |
      |---> Nextcloud Container :80
```

This allows Nextcloud to be opened from another device on the local network using:

```text
http://<VM-IP>:8081
```

Port `8081` shows the Nextcloud service on the Ubuntu VM while port `80` is where the Nextcloud web server is listening inside the container.

## Restart Policy

Both services use:

```yaml
restart: unless-stopped
```

This tells Docker to restart the containers if they stop unexpectedly. It also allows the containers to start again after the Ubuntu VM or Docker restarts, unless the containers were stopped intentionally. This is useful because Nextcloud is supposed to stay available without requiring the containers to be manually started every time the server reboots.

## Checking the Containers

The status of the Nextcloud Compose services can be checked with:

```bash
sudo docker compose ps
```

This shows if the Nextcloud and PostgreSQL containers are running.

The logs can be checked using:

```bash
sudo docker compose logs

sudo docker compose logs postgres
```