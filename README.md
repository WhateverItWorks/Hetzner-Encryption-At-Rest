# my-hetzner-encryption-at-rest
Encryption at rest is encryption that is used to help protect data that is stored on a disk (including solid-state drives) or backup media.

```
CRYPTPASSWORD changethis
DRIVE1 /dev/sda # change this to your own drive
DRIVE2 /dev/sdb # change this to your own drive and this is for if you have multiple drives on your server
SWRAID 1 # Disk mirroring, also known as RAID 1, is the replication of data to two or more disks. Disk mirroring is a good choice for applications that require high performance and high availability, such as transactional applications, email and operating systems.
SWRAIDLEVEL 1 # Disk mirroring, also known as RAID 1, is the replication of data to two or more disks. Disk mirroring is a good choice for applications that require high performance and high availability, such as transactional applications, email and operating systems.
BOOTLOADER grub
HOSTNAME changethis # good to change your hostname as well and also setup firewall to block bad scrapers like censys, shodan, etc
PART /boot ext4 1024M # i this to 1gb, but you can change it if you want, this for unencrypted data when you need to enter your full disk encryption password
PART /     ext4 all crypt # leave, unless you know what your doing
IMAGE /root/images/Ubuntu-2004-focal-64-minimal.tar.gz # leave, unless you know what your doing
SSHKEYS_URL /tmp/authorized_keys # leave, unless you know what your doing


### Skip this step, unless reasons below!

# This is a little help if you want to use LVM (f.e. with Proxmox):
PART /boot ext3 512M
PART lvm   vg0   all crypt

LV vg0 root /    ext4  20G
LV vg0 swap swap swap   4G
```


## Is my data safe on your instances?

Yes, I enable 2FA on my hetzner account [here](https://docs.hetzner.com/accounts-panel/accounts/two-factor-authentication/) and use full disk encryption encryption at rest. [here](https://community.hetzner.com/tutorials/install-ubuntu-2004-with-full-disk-encryption)

## Sources

- [How to install Ubuntu 20.04 with full disk encryption](https://community.hetzner.com/tutorials/install-ubuntu-2004-with-full-disk-encryption)

- [Other Providers like OVH](https://www.reddit.com/r/seedboxes/comments/etw7yo/ovh_sys_full_disk_encryption/)
