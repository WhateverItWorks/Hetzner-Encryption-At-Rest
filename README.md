# my-hetzner-encryption-at-rest
Encryption at rest is encryption that is used to help protect data that is stored on a disk (including solid-state drives) or backup media.

The installimage script in the Hetzner Rescue System provides an easy way to install various Linux distributions.

This tutorial shows how to use installimage to install an encrypted Ubuntu 20.04 system and add remote unlocking via SSH (dropbear) in initramfs stored in a separate /boot partition.

### Prerequisites

- Hetzner account

- Server booted into the Rescue System

- RSA or ECDSA SSH public key

- No private networks attached on Hetzner Cloud


### Step 1 - Create or copy SSH public key

In order to remotely unlock the encrypted system SSH key is required. This key will also be used to later login to the booted system. The dropbear SSH daemon included in Ubuntu 20.04 only supports RSA and ECDSA keys. If you do not have such a key, you need to generate one.

For example to generate a 4096 bit RSA SSH key run:
```
ssh-keygen -t rsa -b 4096
```

Copy the public key to the rescue system, e.g. using scp:

Windows 10/11 (Warning: Don't use stock windows as it is spying invasive, Use something like https://revi.cc for secure, private, and performance):
```
scp C:\Users\changethis\.ssh\id_rsa.pub root@change-to-your-server-ip:/tmp/authorized_keys
```
MacOS:
```
I don't have a MacOS, If anyone wants to contribute in adding it go for it.
```
Linux:
```
scp ~/.ssh/id_rsa.pub root@change-to-your-server-ip:/tmp/authorized_keys
```

### Step 2 - Create or copy installimage config file

When installimage is called without any options, it starts in interactive mode and will open an editor after a distribution image has been selected. After exiting the editor, the installation will proceed and the corresponding configuration is saved as /installimage.conf in the installed system. In this tutorial we will pass such a configuration file to install directly.

Create a file ```nano /tmp/setup.conf``` with the following content or copy it to the server in the Rescue system.

Note: Replace ```changethis``` with a secure password and adjust drive names and partitioning as needed.


```
CRYPTPASSWORD changethis
DRIVE1 /dev/sda # change this to your own drive
DRIVE2 /dev/sdb # change this to your own drive and this is for if you have multiple drives on your server
#DRIVE1 /dev/nvme0n1 # nvme ssd support
#DRIVE2 /dev/nvme1n1 # nvme ssd support
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

> This configuration will install Ubuntu on a single drive (/dev/sda) with a separate unencrypted /boot required for remote unlocking.

### Step 3 - Create or copy post-install script

In order to remotely unlock the encrypted partition, we need to install and add the dropbear SSH server to the initramfs which is stored on the unencrypted /boot partition. This will also trigger the inclusion of dhclient to configure networking, but without any extras. To enable support for Hetzner Cloud, we need to add a hook which includes support for RFC3442 routes.

In order to run these additional steps we need a post-install script for installimage

Create a file ```nano /tmp/post-install.sh``` in the Rescue system with the following content:

```
#!/bin/bash


add_rfc3442_hook() {
  cat << EOF > /etc/initramfs-tools/hooks/add-rfc3442-dhclient-hook
#!/bin/sh

PREREQ=""

prereqs()
{
        echo "\$PREREQ"
}

case \$1 in
prereqs)
        prereqs
        exit 0
        ;;
esac

if [ ! -x /sbin/dhclient ]; then
        exit 0
fi

. /usr/share/initramfs-tools/scripts/functions
. /usr/share/initramfs-tools/hook-functions

mkdir -p \$DESTDIR/etc/dhcp/dhclient-exit-hooks.d/
cp -a /etc/dhcp/dhclient-exit-hooks.d/rfc3442-classless-routes \$DESTDIR/etc/dhcp/dhclient-exit-hooks.d/
EOF

  chmod +x /etc/initramfs-tools/hooks/add-rfc3442-dhclient-hook
}


# Install hook
add_rfc3442_hook

# Copy SSH keys for dropbear
mkdir -p /etc/dropbear-initramfs
cp -a /root/.ssh/authorized_keys /etc/dropbear-initramfs/authorized_keys

# Update system
apt-get update >/dev/null
apt-get -y install cryptsetup-initramfs dropbear-initramfs
```
Important note: make the post-install script executable:
> chmod +x /tmp/post-install.sh

### Step 4 - Start installation

Before starting the installation check again the content of the following files:

- /tmp/authorized_keys - your public SSH key (RSA or ECDSA)

- /tmp/setup.conf - installimage config

- /tmp/post-install.sh - is executable and contains the post-install script

Now you are ready to start the installation with the following command:
```
installimage -a -c /tmp/setup.conf -x /tmp/post-install.sh
```
> Wait until the installation completes and check the ```nano debug.txt``` for any errors.

### Step 5 - Boot installed system

After the installation has finished and any errors are resolved, you can run reboot to restart the server and boot the newly installed system. You can watch the boot process if you have a KVM attached or via remote console on a Cloud instance.

After some time the server should respond to ping. Now login via SSH into dropbear and run ```cryptroot-unlock``` to unlock the encrypted partition(s).

```
ssh root@change-to-your-ip

BusyBox v1.30.1 (Ubuntu 1:1.30.1-4ubuntu6.3) built-in shell (ash)
Enter 'help' for a list of built-in commands.

cryptroot-unlock 
Please unlock disk luks-80e097ad-c0ab-47ce-9302-02dd316dc45c:
```
> If the password is correct the boot will continue and you will automatically be disconnected from the temporary SSH session. After a few seconds you can login to your new system.

## Is my data safe on your instances?

Yes, I enable 2FA (Security Keys + TOTP) on my hetzner account. View more [here](https://docs.hetzner.com/accounts-panel/accounts/two-factor-authentication/), As using this tutorial will make your dedicated server full disk encrypted (encryption at rest). Also some services also use end-to-end encryption as well, which means i also can't see your data. View more [here](https://wikiless.whateveritworks.org/wiki/End-to-end_encryption?lang=en)

### Contributing on AbuseIPDB
<a href="https://www.abuseipdb.com/user/118422" title="AbuseIPDB is an IP address blacklist for webmasters and sysadmins to report IP addresses engaging in abusive behavior on their networks">
	<img src="https://www.abuseipdb.com/contributor/118422.svg" alt="AbuseIPDB Contributor Badge" style="width: 401px;">
</a>

## Sources

- [How to install Ubuntu 20.04 with full disk encryption](https://community.hetzner.com/tutorials/install-ubuntu-2004-with-full-disk-encryption)
- [Other Providers like OVH](https://www.reddit.com/r/seedboxes/comments/etw7yo/ovh_sys_full_disk_encryption/)
- [Hetzner Full Disk Encryption](https://github.com/shawnbarton/hetzner-full-disk-encyption)
- [Disk Encryption Hetzner](https://github.com/TheReal1604/disk-encryption-hetzner)
- [Optimized and Maintenance-free Kubernetes on Hetzner Cloud in one command!](https://github.com/kube-hetzner/terraform-hcloud-kube-hetzner)
