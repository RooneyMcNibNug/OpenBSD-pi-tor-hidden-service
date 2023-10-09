# Hosting an OpenBSD Hidden Service on a Raspberry Pi with OpenBSD

Its the Year of ~Linux on Desktop~ Tor on Pi! Let's do it with  our favorite pufferfish 🐡

---

### Downloading OpenBSD

You can head over to www.openbsd.org to do this - any one of the [mirrors](https://www.openbsd.org/ftp.html) available on there will have an `.img` file of the architecture you need - which, for the raspberry pi, is the `arm64` image.

So go to something like https://cdn.openbsd.org/pub/OpenBSD/7.3/arm64/ (or whatever the latest version number is for you as of reading this, replacing the URL accordingly) and download the `miniroot73.img` file, which is nice and small.

### Imaging OpenBSD on a microSD

This guide assumes you have a *nix machine or Windows with [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) installed.

⚠️ BE CAREFUL / BE SURE THAT YOU ARE IMAGING TO THE CORRECT DISK, AS NOT TO WIPE YOUR PRIMARY OS OR ANYTHING ELSE ⚠️

We can easily use `dd` for this from the terminal. `cd` to the directory where you saved the OpenBSD image, then identify the microSD with something like `df -h` or `lsblk`, then run the `dd` command to write the image to the disk:

```console
$ cd /home/user/Downloads
$ lsblk -o NAME,MOUNTPOINT,SIZE,VENDOR,MODEL
NAME   MOUNTPOINT  SIZE VENDOR   MODEL
sda                500G Seagate  XYZ
├─sda1 /boot       200M 
├─sda2 [SWAP]      4G 
└─sda3 /           496G
sdb                24G  Kingston SD

$ dd if=miniroot73.img of=/dev/sdb bs=1M conv=fsync
```

You should see something like `2511872 bytes transferred in 0.034 secs (72095271 bytes/sec)` as the output. At that point, you can eject the microSD and insert it into your Raspberry Pi.

### Installing OpenBSD onto a RaspberryPi 3B+ (or other model)

For the installation process on the Pi, we will need to get a shell via USB Serial. This might be unfamiliar territory for some, but with caution and the right tools we can make it happen.

You will want to have either a USB to TTL Serial Cable like [this](https://www.adafruit.com/product/954), or something like a [FTDI Friend](https://learn.adafruit.com/ftdi-friend/overview) adapter. In this example, I will use the FTDI Friend, but regardless you need to make sure you understand the pin locations and what they do - your best bet for that is to first consult this website: https://pinout.xyz



![image](https://github.com/RooneyMcNibNug/OpenBSD-pi-tor-hidden-service/assets/17930955/c18c2a11-c45f-4891-a3f5-df0176eac462)

![image](https://github.com/RooneyMcNibNug/OpenBSD-pi-tor-hidden-service/assets/17930955/784921c3-a03a-4f4d-997c-142932fe5095)


### Initial steps afterboot

![image](https://github.com/RooneyMcNibNug/OpenBSD-pi-tor-hidden-service/assets/17930955/8a074641-deec-4b59-963a-0ed037308671)


### Installing tor and setting up the hidden service

## "Alright, what can I do with this thing?

### Serving up a site with HTTPD

### using the server as a unix ssh box over tor

One of the many great things about a tor hidden service is that it accomplishes NAT traversal automatically through the tor network, allowing you to connect to the host here from ther road, and even other resources on the network it is being hosted within from that point. WHo needs a VPN when you have a tor hidden service running on your Pi!

In order to accomplish this, you need to add the following to your `/etc/tor/torrc` config:


```
HiddenServicePort 22 127.0.0.1:22
```

After adding this, you'll want to do restart tor with a quick `rcctl restart tor`.

You may have noticed earlier that we used OpenBSD's package manager to install [torsocks](https://gitlab.torproject.org/tpo/core/torsocks/) - we can use this here on our local machine too, so make sure ti install it on your desktop/laptop you will remote into the Pi with. 

From there, you can do the following:

```console
user@remote_pc:~ $ torsocks ssh user@[site].onion
The authenticity of host ‘[site].onion (127.42.42.0)' can't be established.
ECDSA key fingerprint is SHA256:blahblahblah.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added ‘[site].onion' (ECDSA) to the list of known hosts.
user@[site].onion's password:
Last login: Thu Sep 21 15:28:46 2023 from 192.168.0.x
OpenBSD 7.3 (GENERIC.MP) #1: Mon Jul 24 12:12:28 MDT 2023

Welcome to OpenBSD: The proactively secure Unix-like operating system.

Please use the sendbug(1) utility to report bugs in the system.
Before reporting a bug, please try to reproduce it with the latest
version of the code.  With bug reports, please try to ensure that
enough information to reproduce the problem is enclosed, and if a
known fix for it exists, include that as well.

openbsdeez$
```

### hosting a git repo

Download arm64 img
Install with GPIO pins and
Install non-root user
Install all sets
Setup doas for non-root user
Get things updated: systpatch, pkg_add -u
pkg_add -i tor torsocks vim exfat-fuse htop
mkdir Documents && cp /etc/tor/torrc home/user/Documents/torrc.backup
Uncomment the tor dir lines and port 80 lins

### READING MATERIALS:

'How do Onion Services work?' - https://community.torproject.org/onion-services/overview/
Openbsd's manual page for HTTPD.conf - https://man.openbsd.org/httpd.conf.5
