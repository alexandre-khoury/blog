# How to install Arch Linux on a Hetzner VPS

## Easy (fast) way
1. Create a new server (choose any distribution at this point, it doesn't matter)
1. In your new server's settings, go into `Rescue > Enable rescue & power cycle` (choose `linux64` as OS)
1. After a few seconds, you can log in via SSH using the credentials shown
1. Run `installimage`
1. Select Arch Linux
1. When shown the installation configuration file, exit with `F10` and confirm
1. `reboot`

You can now log in again via SSH.
Possibly you will need to remove the host key in your `known_hosts` by running `ssh-keygen -R [ip_address]`

## Advanced (custom) way
The first method is nice when you want to get things going quickly, but all the magic is hidden in [installimage](https://github.com/hetzneronline/installimage). This is not in the spirit of the distro, so let's now set up Arch Linux 100% manually!

### Step 1: Boot in rescue mode
We want to be able to write to the main disk, so there is no other way than in rescue mode. Follow the first 3 steps of the [Easy (fast) way](#easy-fast-way). Take note of the following information shown on boot:

```
Network data:
   eth0  LINK: yes
         MAC:  [MY_MAC]
         IP:   [MY_IPv4]
         IPv6: [MY_IPv6]
         Virtio network driver
```

### Step 2: Follow the wiki

Follow the steps [here](https://wiki.archlinux.org/title/Install_Arch_Linux_from_existing_Linux#Method_A:_Using_the_bootstrap_image_(recommended)). The code below is a summary of the commands, with the 5th line not in the wiki:

```
cd /tmp
wget mirror_link -O bootstrap.tar.gz
tar xzf bootstrap.tar.gz --numeric-owner
nano /tmp/root.x86_64/etc/pacman.d/mirrorlist # Uncomment mirrors
mount --bind /tmp/root.x86_64/ /tmp/root.x86_64/ ### /!\ Important! /!\ ###
/tmp/root.x86_64/bin/arch-chroot /tmp/root.x86_64/
pacman-key --init
pacman-key --populate archlinux
pacman -Syyu
```

### Step 3: Configure networking

You can now follow the [usual steps](https://wiki.archlinux.org/title/installation_guide) of installing your custom system.

Don't forget to:

- Install `openssh`
- Enable `sshd.service` and `systemd-networkd.service`
- Set up a root password with `passwd`
- Set `PermitRootLogin yes` in `/etc/ssh/sshd_config`
- Copy [10-eth0.link](./10-eth0.link) and [10-eth0.network](./10-eth0.network) to `/etc/systemd/network/` after modifying the template with your values

FYI, the network configuration is based on the official documentation:
- https://docs.hetzner.com/robot/dedicated-server/network/network-configuration-using-systemd-networkd
- https://docs.hetzner.com/cloud/servers/static-configuration/
