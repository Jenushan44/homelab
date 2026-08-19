# Nextcloud Deployment

## Overview

Nextcloud is deployed on an Ubuntu Server virtual machine running in my Proxmox homelab. I set it up to create a self-hosted cloud storage environment where files can be stored on my own server instead of relying on a third-party cloud storage. The deployment uses Docker and Docker Compose to run Nextcloud and PostgreSQL in separate containers. Nextcloud provides the web interface and handles file management, users, folders and other cloud storage features. PostgreSQL stores the structured application data used by Nextcloud. Docker volumes are used to keep the Nextcloud files and PostgreSQL database persistent even if the containers are stopped, removed, or recreated.

## Architecture

The deployment uses two main Docker containers:

- **Nextcloud container** - runs the Nextcloud application and web interface.
- **PostgreSQL container** - runs the PostgreSQL database used by Nextcloud.

Both containers are managed together using Docker Compose.

```text
Client
  |
  | HTTP
  |
  | VM-IP:8081
  |
  |---> Ubuntu Server VM
            |
            | 8081:80
            |
            |
            |--->  Nextcloud Container
                          |
                          | Database Connection
                          |
                          |---> PostgreSQL Container
```

The containers also use separate Docker volumes for persistent storage:

```text
Nextcloud Container
       |
       |--->nextcloud_data
                |
                |---> Nextcloud installation and uploaded files


PostgreSQL Container
       |
       |---> postgres_data
                 |
                 |---> PostgreSQL database files
```

## Technology Stack

The deployment uses:

- **Proxmox VE** to host the Ubuntu Server virtual machine.
- **Ubuntu Server** as the operating system running Docker.
- **Docker** to run Nextcloud and PostgreSQL in separate containers.
- **Docker Compose** to configure and manage the containers together.
- **Nextcloud** for the cloud storage application.
- **PostgreSQL** for Nextcloud's database.
- **Docker volumes** for persistent storage.

## Docker Compose

Docker Compose is used to define and manage the Nextcloud and PostgreSQL services.

The deployment uses these Docker images:

```text
nextcloud:latest
postgres:16
```

The Compose configuration also defines:

- environment variables
- persistent volumes
- port mapping
- container restart policies
- communication between the containers

The containers can be started together using:

```bash
sudo docker compose up -d
```

The `-d` option runs the containers in detached mode so that they continue running in the background after the terminal is closed.

Both services use:

```yaml
restart: unless-stopped
```

This allows the containers to start again automatically after Docker or the Ubuntu Server VM restarts unless they were intentionally stopped.

## PostgreSQL Database

Nextcloud uses PostgreSQL to store the structured application data.

The PostgreSQL service receives its database configuration through environment variables:

```text
POSTGRES_DB
POSTGRES_USER
POSTGRES_PASSWORD
```

Nextcloud receives the same database information with:

```text
POSTGRES_HOST=postgres
```

The hostname `postgres` works because both containers are connected through the Docker Compose network. Docker can resolve the PostgreSQL service name to the PostgreSQL container. PostgreSQL uses port `5432` internally, but this port is not published to the Ubuntu VM because the database only needs to be accessed by Nextcloud inside the Docker environment.

## Network Access

Nextcloud's web server listens on port `80` inside its container.

Docker maps port `8081` on the Ubuntu Server VM to port `80` inside the Nextcloud container:

```text
8081:80
```

The flow for the request is:

```text
Browser
   |
   | http://<VM-IP>:8081
   |
   |---> Ubuntu Server :8081
                |
                |
                | Docker port mapping
                |
                |---> Nextcloud Container :80
```

This allows Nextcloud to be accessed from another device on the local network using:

```text
http://<VM-IP>:8081
```

The PostgreSQL container does not need a published host port because Nextcloud communicates with it through the internal Docker Compose network.

## Persistent Storage

Containers should not be used as permanent storage because containers can be removed and recreated.

Two Docker volumes are used:

```text
nextcloud_data
postgres_data
```

The Nextcloud volume is mounted to:

```text
/var/www/html
```

The PostgreSQL volume is mounted to:

```text
/var/lib/postgresql/data
```

This separates the persistent data from the lifecycle of the containers.

```text
Container
   |
   | can be removed
   |
   |---> Recreated Container
                |
                | reconnects to
                |
                |---> Persistent Volume
```

## Persistence Testing

After completing the initial Nextcloud setup, I tested whether or not the deployment actually preserved its data when the containers were removed.

I uploaded a test file to Nextcloud and then removed the containers using:

```bash
sudo docker compose down
```

The Docker volumes were not removed so I recreated the containers using:

```bash
sudo docker compose up -d
```

After the containers started again, Nextcloud still recognized the administrator account and the uploaded test file was still available. This confirmed that the application data was being stored in the persistent Docker volumes instead of depending on the containers themselves.

## Troubleshooting

During the first deployment, PostgreSQL stopped running and Nextcloud returned an internal server error.

The PostgreSQL logs contained errors such as:

```text
No space left on device
```

Checking the Ubuntu Server filesystem showed that the root filesystem was completely full. The Proxmox virtual disk had 32 GB available, but the Ubuntu volume was only using 15 GB. The remaining space was already available inside the LVM volume group but had was not assigned to the root logical volume.

I checked the storage layout using:

```bash
df -h
lsblk
sudo vgs
sudo lvs
```

The logical volume was changed to use the remaining free space:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

This increased the root filesystem from approximately 15 GB to 30 GB. Then, PostgreSQL was then able to start normally and Nextcloud became accessible again.

## V1 Result

The first version of the Nextcloud deployment provides a working self-hosted cloud storage environment on the local network.

V1 includes:

- Nextcloud running in Docker
- PostgreSQL running in Docker
- Docker Compose service management
- environment variables for database configuration
- internal Docker networking
- local network access through port `8081`
- persistent Nextcloud storage
- persistent PostgreSQL storage
- automatic container restart policies
- tested container recreation and data persistence

## V2 - Backup and Recovery

Version 2 adds backups so that the Nextcloud data is not only protected by Docker volumes. I added a separate 1 TB HDD to the Proxmox server for backup storage. The HDD is mounted on Proxmox and added as a storage location called `backup-hdd`.

Then, I added a 500 GiB virtual disk from this storage to the Ubuntu Server VM and mounted it at:

```text
/mnt/backups
```

The backup storage setup is:

```text
1 TB HDD
   |
   |---> Proxmox /mnt/backups
             |
             |---> backup-hdd
                       |
                       |---> 500 GiB Virtual Disk
                                  |
                                  |---> Ubuntu /mnt/backups
```

This storage will be used to keep backups of the PostgreSQL database and the Nextcloud files.

## Manual Backups

After setting up the backup storage, I tested creating backups of both parts of the Nextcloud deployment.

### PostgreSQL Backup

I used `pg_dump` to create a backup of the PostgreSQL database.

The backup was saved as:

```text
/mnt/backups/postgres/nextcloud.sql
```

`pg_dump` creates a logical backup of the database which includes the database state that Nextcloud needs such as users, file metadata, settings and other application information.

### Nextcloud Data Backup

I also backed up the `nextcloud_nextcloud_data` Docker volume. This volume has the Nextcloud files and data. First, I put Nextcloud into maintenance mode so that its data would not change while the backup was being created.

Then, the volume was packaged into a compressed `.tar.gz` archive and saved as:

```text
/mnt/backups/nextcloud/nextcloud-data.tar.gz
```

After the backup finished, I turned maintenance mode back off.

The backup setup now looks like:

```text
/mnt/backups/
|
|-- postgres/
|     |
|     |--> nextcloud.sql
|
|--> nextcloud/
      |
      |--> nextcloud-data.tar.gz
```

Both of the backups were checked to make sure that they were created and contained data. These are currently manual backups and they only have the data from the time they were created. The next step is to automate this process so that new backups can be created automatically.

## Future Improvements

- backups and restore testing
- HTTPS
- Nginx reverse proxy
- secure remote access through a VPN
- dedicated storage for personal files
- monitoring and alerts
- additional Nextcloud security hardening
- multi-factor authentication