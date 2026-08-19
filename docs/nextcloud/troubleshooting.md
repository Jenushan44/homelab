# Troubleshooting

This document goes over the main problem that I experience while setting up the Nextcloud deployment.

## PostgreSQL Failed Because the Ubuntu Disk Was Full

### Problem

After starting the Nextcloud and PostgreSQL containers, the Nextcloud web page returned:

```text
Internal Server Error
```

The Nextcloud container was still running, but the PostgreSQL container was stopped so I checked the PostgreSQL logs using:

```bash
sudo docker compose logs postgres
```

The logs showed errors including:

```text
No space left on device
```

PostgreSQL could not write the files it needed to write to in order to run.

### Checking Disk Space

I checked the Ubuntu Server filesystem using:

```bash
df -h
```

The root filesystem showed:

```text
15 GB total
15 GB used
0 GB available
100% used
```

This showed why PostgreSQL was not able to start. Docker images, containers and build cache were using a lot of the available storage.

I also checked Docker's storage usage using:

```bash
sudo docker system df
```

Some unused Docker build cache was removed, but this did not give me enough free space to solve the problem.

### Checking the Proxmox Virtual Disk

I checked the Ubuntu VM configuration in Proxmox and found out that the virtual disk was already configured with:

```text
32 GB
```

This showed that the virtual disk did not need to be increased so my next step was checking how Ubuntu was using the available virtual disk space.

### Checking the Ubuntu Storage Layout

I used these commands to inspect the Ubuntu disk and configuration:

```bash
lsblk
sudo vgs
sudo lvs
```

The virtual disk was 32 GB and the Ubuntu volume group had about 30 GB. However, the root logical volume was only using about 15 GB. The 15 GB left was available as unused space inside the LVM volume group.

Storage Layout:

```text
Proxmox Virtual Disk
        |
        | 32 GB
        |
        |---> Ubuntu Disk
                  |
                  |
                  |---> Ubuntu LVM Volume Group
                                |
                                |-- 15 GB -> Root Filesystem
                                |
                                |-- 15 GB -> Unused LVM Space
```

### Expanding the Logical Volume

Instead of increasing the Proxmox disk, I increased the Ubuntu root logical volume that was already there to use the free space that was available in the volume group.

I used:

```bash
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

The command extended the volume, used all free space leftover in the volume group and resized the filesystem to use the extra storage. 

After the change, I checked the filesystem again using:

```bash
df -h
```

The root filesystem increased from approximately 15 GB to approximately 30 GB. The usage dropped from 100% to 50%.

### Result

After expanding the filesystem, I restarted the Nextcloud Compose stack:

```bash
sudo docker compose up -d
```

The PostgreSQL container started successfully and then Nextcloud loaded normally and showed the administrator account setup screen. The issue showed that the VM had enough virtual disk capacity in Proxmox, but Ubuntu was not using all of the available space because part of the LVM volume group had was not assigned to the root filesystem.

The troubleshooting process was:

```text
Nextcloud Internal Server Error
        |
        |---> Check Containers
                    |
                    |---> PostgreSQL Stopped
                                  |
                                  |---> Check PostgreSQL Logs
                                                |
                                                |---> "No space left on device"
                                                                  |
                                                                  |---> Check df -h
                                                                            |
                                                                            |---> Root Filesystem 100% Full
                                                                                            |
                                                                                            |---> Check Promox + LVN
                                                                                                          |
                                                                                                          |---> Unused LVM Space found
                                                                                                                        |
                                                                                                                        |---> Exteneded the root logical volume
                                                                                                                                            |
                                                                                                                                            |---> PostgreSQL and Nextcloud working
```

## Backup Disk Failed to Mount Using UUID

### Problem

After adding the backup disk to Ubuntu, I added it to `/etc/fstab` using its UUID so that it would mount automatically.

When I tested the configuration using:

```bash
sudo mount -a
```

I received an error saying:

```text
Can't lookup blockdev
special device UUID=... does not exist
```

### Checking the Disk

I first checked that the disk and its UUID existed:

```bash
sudo blkid /dev/sdb
```

The disk was still using ext4 and had the same UUID that I added to `/etc/fstab`.

I also checked the UUID mapping using:

```bash
ls -l /dev/disk/by-uuid/
```

This showed that the UUID correctly pointed to:

```text
/dev/sdb
```

Then, I mounted the disk directly using:

```bash
sudo mount /dev/sdb /mnt/backups
```

This worked, which showed that the disk, filesystem and mount point were working and the problem was with how the disk was being found through `/etc/fstab`.

### Fix

Instead of using:

```text
UUID=<backup-disk-uuid>
```

I used the UUID device path:

```text
/dev/disk/by-uuid/<backup-disk-uuid>
```

in `/etc/fstab`.

Then, I tested the configuration again using:

```bash
sudo umount /mnt/backups
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/backups
```

The disk mounted successfully at `/mnt/backups`.

### Result

The backup disk can now be mounted automatically using the `/etc/fstab` configuration.

The main troubleshooting process was:

```text
mount -a failed
      |
      |---> Check disk UUID
                |
                |---> UUID exists
                          |
                          |---> Check UUID mapping
                                    |
                                    |---> UUID points to /dev/sdb
                                              |
                                              |---> Test direct mount
                                                        |
                                                        |---> Disk mounts successfully
                                                                  |
                                                                  |---> Use /dev/disk/by-uuid/ path
                                                                            |
                                                                            |---> mount -a works
```