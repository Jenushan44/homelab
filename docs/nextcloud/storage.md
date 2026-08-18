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

## Backup Storage

For Version 2, I added a separate 1 TB HDD that will be used to store backups. The main SSD is being used by Proxmox and the Ubuntu Server VM, so using another physical drive gives me a separate place to store backup data. The HDD originally used NTFS, which is mainly used with Windows and because Proxmox is Linux based, I changed the filesystem on the main HDD partition to ext4.

The HDD on Proxmox showed up as:

```text
/dev/sdb
```

and the partition I used was:

```text
/dev/sdb2
```

I created the ext4 filesystem using:

```bash
mkfs.ext4 /dev/sdb2
```

## Mounting the HDD

After creating the filesystem, I needed a way to access it through the Linux directory tree.

I created:

```text
/mnt/backups
```

and mounted `/dev/sdb2` to it:

```bash
mount /dev/sdb2 /mnt/backups
```

This means that when I access `/mnt/backups` on Proxmox, I am accessing the filesystem stored on the HDD.

```text
1 TB HDD
   |
   |---> /dev/sdb2
             |
             |---> ext4
                     |
                     |---> /mnt/backups
```

I also added the mount to `/etc/fstab` using the filesystem UUID. This makes the mount persistent so that Proxmox can mount the HDD again after a reboot.

## Proxmox Backup Storage

Mounting the HDD makes it available to Linux, but my Ubuntu VM is isolated from the Proxmox host and cannot directly see the HDD.

I added `/mnt/backups` to Proxmox as a storage location called:

```text
backup-hdd
```

I then created a 500 GiB virtual disk using `backup-hdd` and attached it to my Ubuntu Server VM. A virtual disk looks like a normal disk to the VM, but Proxmox controls where its storage actually comes from.

```text
1 TB HDD
   |
   |---> Proxmox /mnt/backups
             |
             |---> backup-hdd
                       |
                       |---> 500 GiB Virtual Disk
                                  |
                                  |---> Ubuntu VM
```

Ubuntu sees this virtual disk as:

```text
/dev/sdb
```

This is not the same `/dev/sdb` as the physical HDD on Proxmox. Proxmox and Ubuntu have their own views of the storage devices that are available to them.

## Mounting the Backup Disk in Ubuntu

The new virtual disk was blank, so I created an ext4 filesystem on it:

```bash
sudo mkfs.ext4 /dev/sdb
```

Then, I created another `/mnt/backups` mount point and this time, it was inside the Ubuntu VM:

```bash
sudo mkdir -p /mnt/backups
```

and then mounted the virtual disk:

```bash
sudo mount /dev/sdb /mnt/backups
```

I also added this mount to Ubuntu's `/etc/fstab` so that it will still be available after the VM reboots.

The final storage setup is:

```text
1 TB HDD
   |
   |---> Proxmox /dev/sdb2
             |
             |---> Proxmox /mnt/backups
                       |
                       |---> backup-hdd
                                 |
                                 |---> 500 GiB Virtual Disk
                                            |
                                            |---> Ubuntu /dev/sdb
                                                       |
                                                       |---> Ubuntu /mnt/backups
```

The Ubuntu `/mnt/backups` directory will be used to store the Nextcloud and PostgreSQL backups.