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
Possibly after removing the host key in your `known_hosts` with `ssh-keygen -R [ip_address]`
## Advanced (custom) way
The first method is nice when you want to get things going quickly, but all the magic is hidden in [installimage](https://github.com/hetzneronline/installimage). This is not in the spirit of the distro, so let's now set up Arch Linux 100% manually!
