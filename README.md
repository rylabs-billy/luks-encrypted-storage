# Luks Encrypted Storage

This is a POC that tests:
- Encrypting a data partition (/dev/sdc) with Luks
- Expanding the Luks container to a [Linode Block Storage](https://www.linode.com/docs/guides/platform/block-storage/) Volume (external device).

In this example we've partitioned a Linode's allocable space into three disks: `root`, `swap`, and `data`.

- `/dev/sda` - Debian 10 Disk
- `/dev/sdb` - 512 MB Swap Image
- `/dev/sdc` - Data

Install required packages and load the `dm-crypt` kernel module.
```
apt install cryptsetup lvm2 -y
modprobe dm-crypt
```

Encrypt the data partition (`/dev/sdc`) - create the physical volume, volume group, and logical volume.
```
pvcreate /dev/sdc
vgcreate vg_data /dev/sdc
lvcreate -l 100%FREE -n lv_data vg_data
```
Sanity check:
```
lsblk
...
NAME                MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                   8:0    0  9.8G  0 disk /
sdb                   8:16   0  512M  0 disk [SWAP]
sdc                   8:32   0 39.8G  0 disk
└─sdc1                8:33   0 39.8G  0 part
  └─vg_data-lv_data 254:0    0 39.7G  0 lvm
```
Create a Luks container in the logical volume and then open it. You can label the container anything you want. In this example we're labeling it `crypt_data` to make it easily distiguishable.
```
cryptsetup luksFormat --hash=sha512 --key-size=512 --cipher=aes-xts-plain64 --verify-passphrase /dev/vg_data/lv_data
cryptsetup luksOpen /dev/vg_data/lv_data crypt_data
```
Make a filesystem in `crypt_data` and mount it. In this example we're mounting it to the `/opt` directory.
```
mkfs.ext4 /dev/mapper/crypt_data
mount /dev/mapper/crypt_data /opt
```
Sanity check:
```
lsblk
...
NAME              MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda                 8:0    0  9.8G  0 disk  /
sdb                 8:16   0  512M  0 disk  [SWAP]
sdc                 8:32   0 39.8G  0 disk
└─vg_data-lv_data 254:0    0 39.7G  0 lvm
  └─crypt_data    254:1    0 39.7G  0 crypt /opt
```
Now let's make things more interesting by expanding the Luks container to span across another disk, as one logical container. We'll start by [attaching](https://www.linode.com/docs/guides/how-to-use-block-storage-with-your-linode/) a Linode Block Storage volume to `/dev/sdd`:

- `/dev/sda` - Debian 10 Disk
- `/dev/sdb` - 512 MB Swap Image
- `/dev/sdc` - Data
- `/dev/sdd` - Linode BS Volume

Resize the VG (vg_data), LV (lv_data), Luks container (crypt_data),and ext4 filesystem to include the Block Storage volume.

Extend the VG:
```
pvcreate /dev/sdd
vgextend vg_data /dev/sdd
```

Extend the LV:
```
umount /dev/mapper/crypt_data
e2fsck -f /dev/mapper/crypt_data
cryptsetup luksClose /dev/mapper/crypt_data
lvextend -l +100%FREE -n vg_data/lv_data
```

Extend the Luks container:
```
cryptsetup luksOpen /dev/vg_data/lv_data crypt_data
umount /dev/mapper/crypt_data
cryptsetup --verbose resize crypt_data
```

Extend the filesystem:
```
e2fsck -f /dev/mapper/crypt_data
resize2fs /dev/mapper/crypt_data
mount /dev/mapper/crypt_data /opt
```
Santiy check:
```
lsblk
...
NAME              MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda                 8:0    0  9.8G  0 disk  /
sdb                 8:16   0  512M  0 disk  [SWAP]
sdc                 8:32   0 39.8G  0 disk
└─vg_data-lv_data 254:0    0 59.7G  0 lvm
  └─crypt_data    254:1    0 59.7G  0 crypt /opt
sdd                 8:48   0   20G  0 disk
└─vg_data-lv_data 254:0    0 59.7G  0 lvm
  └─crypt_data    254:1    0 59.7G  0 crypt /opt
```

Now the `/dev/sdc` (39.8GB) and `/dev/sdd` (20GB) make one logical volume that is 59.7GB. We can see this in action by creating a file that's larger than the individual disks, but fits within the size of the container. 

```
fallocate -l 50G /opt/test
ls -lah /opt/test
...
-rw-r--r-- 1 root root 50G Jan 15 20:33 /opt/test
```
```
df -h
...
Filesystem              Size  Used Avail Use% Mounted on
udev                    982M     0  982M   0% /dev
tmpfs                   200M   21M  180M  11% /run
/dev/sda                9.6G  1.3G  7.9G  14% /
tmpfs                   998M     0  998M   0% /dev/shm
tmpfs                   5.0M     0  5.0M   0% /run/lock
tmpfs                   998M     0  998M   0% /sys/fs/cgroup
/dev/mapper/crypt_data   59G   51G  5.5G  91% /opt        
tmpfs                   200M     0  200M   0% /run/user/0
```

