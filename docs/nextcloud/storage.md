# Storage and Persistence

## Overview

The Nextcloud deployment uses persistent Docker volumes so that important data is stored separately from the containers. Containers can be stopped, removed and recreated, so important files should not depend on the container's own filesystem.

The deployment uses two named volumes and they store different types of data:

```text
nextcloud_data
postgres_data
```

## Nextcloud File Storage

The `nextcloud_data` volume is attached to the Nextcloud container.

```yaml
volumes:
  - nextcloud_data:/var/www/html
```

The left side is the Docker managed volume:

```text
nextcloud_data
```

The right side is the directory inside the Nextcloud container where the persistent Nextcloud application data is stored:

```text
/var/www/html
```

It includes the files used by the Nextcloud installation and the uploaded user files.

```text
Nextcloud Container
        |
        |
        |---> /var/www/html
                    |
                    |---> nextcloud_data Volume
```

The volume is separate from the container and if the Nextcloud container is removed and recreated, the new container can reconnect to the same `nextcloud_data` volume.

## PostgreSQL Storage

The PostgreSQL container uses a different volume:

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

PostgreSQL stores its database files inside:

```text
/var/lib/postgresql/data
```

Docker connects that directory to the `postgres_data` volume.

```text
PostgreSQL Container
        |
        |--> /var/lib/postgresql/data
                        |
                        |---> postgres_data Volume
```

The database has application data that Nextcloud needs to operate and includes information related to users, settings, file metadata, sharing and other Nextcloud application data.

## Database Storage vs File Storage

Nextcloud uses both PostgreSQL and normal file storage because they are used for different types of information. PostgreSQL stores structured application data and Nextcloud file storage contains the actual uploaded files.

For example, when a photo is uploaded:

```text
                    Nextcloud
                    /       \
                   /         \
                  /           \
           PostgreSQL     File Storage
           structured     actual file
           information     example.png
```

The actual image is stored in the Nextcloud file storage. PostgreSQL stores the structured information Nextcloud needs to manage the application and its files. Both of them are not backup and they are both apart of the main Nextcloud system.

## Why Persistent Storage Is Needed

If important files were stored only inside a container, removing the container could remove that data with it.

Without a volume:

```text
Container
   |
   | contains file
   |
   |---> example.png

Container removed
   |
   |---> example.png lost
```

With a volume:

```text
Container
   |
   |
   |---> Persistent Volume
                |
                |
                |---> example.png
```

Then, the container can be removed:

```text
Old Container
     X

Persistent Volume
      |
      |---> example.png stays
```

A new container can reconnect to the volume.

## Persistence Test

I tested the storage configuration instead of only assuming that the volumes were working. First, I uploaded a test file through the Nextcloud web interface.

Then, I then removed both of the containers using:

```bash
sudo docker compose down
```

The named volumes were not removed and the containers were recreated using:

```bash
sudo docker compose up -d
```

After Nextcloud started again:

- the administrator account still existed
- the uploaded test file was still available

This confirmed that the important data was stored in the persistent volumes instead of depending on the original containers.

Result:

```text
Containers Removed
      |
      |---> Volumes Stay
                  |
                  |
                  |---> Containers Recreated
                                |
                                |---> Existing Data Restored
```

The administrator account and other database information also did not get deleted through the PostgreSQL volume and the uploaded file did not get deleted through the Nextcloud volume.

## Persistence Is Not a Backup

Docker volumes protect the data from the normal container behavior but they do not protect against every type of failure. For example, if the physical storage device that has the volumes fails, the volumes could also be lost.

```text
Container deleted
      |
      |---> Volume remains


Physical disk fails
      |
      |---> Volume could be lost
```

This is why backups are planned as the next improvement to the Nextcloud deployment. A backup will give another copy of the files and database data outside of the main Docker volumes.