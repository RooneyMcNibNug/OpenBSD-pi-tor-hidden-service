# Hosting an OpenBSD Hidden Service on a Raspberry Pi with OpenBSD

Its the Year of ~Linux on Desktop~ Tor on Pi! Let's do it with our favorite pufferfish 🐡

---

### Downloading OpenBSD

You can head over to www.openbsd.org to do this - any one of the [mirrors](https://www.openbsd.org/ftp.html) available on there will have an `.img` file of the architecture you need - which, for the raspberry pi, is the `arm64` image.

So go to something like https://cdn.openbsd.org/pub/OpenBSD/7.6/arm64/ (or whatever the latest version number is for you as of reading this, replacing the URL accordingly) and download the `miniroot76.img` file, which is nice and small.

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

$ dd if=miniroot76.img of=/dev/sdb bs=1M conv=fsync
```

You should see something like `2511872 bytes transferred in 0.034 secs (72095271 bytes/sec)` as the output. At that point, you can eject the microSD and insert it into your Raspberry Pi.

###  Preparing and accessing the RaspberryPi 3B+ (or other model)

For the installation process on the Pi, we will need to get a shell via USB Serial. This might be unfamiliar territory for some, but with caution and the right tools we can make it happen.

You will want to have either a USB to TTL Serial Cable like [this](https://www.adafruit.com/product/954), or something like a [FTDI Friend](https://learn.adafruit.com/ftdi-friend/overview) adapter. In this example, I will use the FTDI Friend, but regardless you need to make sure you understand the pin locations and what they do - your best bet for that is to first consult this website: https://pinout.xyz

![image](https://github.com/RooneyMcNibNug/OpenBSD-pi-tor-hidden-service/assets/17930955/c18c2a11-c45f-4891-a3f5-df0176eac462)

![image](https://github.com/RooneyMcNibNug/OpenBSD-pi-tor-hidden-service/assets/17930955/784921c3-a03a-4f4d-997c-142932fe5095)

With the Serial to USB trinket all plugged into the Pi one end and your laptop or PC on the other, use the following to list all connected devices and look for the one corresponding to your Pi - it will usually be something like `/dev/ttyUSB0` or `/dev/ttyACM0`:

```
dmesg | grep -i "tty"
```

(You may need to use `sudo` here.)

Take note of the output. For this example, lets say its `/dev/ttyUSB0`. On your *nix OS, changes are you already have the `screen` application installed. We will need this, so make sure it is. If you need to install it, please use your system's package manager to do so, as it is available on almost all distros: https://pkgs.org/download/screen

Now in the terminal get ready to connect to the Pi:

```
screen /dev/ttyUSB0 115200
```

(The numbers at the end is the baud rate, but let's not worry about that so much right now)

You'll get some stuff on the screen, but mostly you are looking to make sure you see the following:

```
Welcome to the OpenBSD/amd64 7.0 installation program.
(I)nstall, (U)pgrade, (A)utoinstall or (S)hell?
```
# Installing OpenBSD on the Pi

Now, its time for the install. This can seem a bit daunting, but following https://www.openbsd.org/faq/faq4.html#Install will get you easily on your way, I promise. It is recommended that you install all Sets during this process.

Here are some other context-specific notes on the install here:

- Make sure to use the `?` option for the interface configuration options to find the best valid interface if you are using ethernet (which you should)
- It is a very good idea to encrypt the root disk with a passphrase during the installation, which is now offered as a step during disks setup

### Initial steps afterboot

Yay, we have OpenBSD installed on our Pi! Celebrate, and then when you are done, there are a few important things we should do upon first boot.

OpenBSD has their https://man.openbsd.org/afterboot as a nice checklist, but allow me to list out some thing here:

1. change the root password with `passwd root`
2. If you want to change your system's hostname, you can  `vim /etc/myname`
3. Its worth just double-checking that there aren't any missing patches by running `syspatch`

![image](https://github.com/RooneyMcNibNug/OpenBSD-pi-tor-hidden-service/assets/17930955/8a074641-deec-4b59-963a-0ed037308671)


### Installing tor and setting up the hidden service

Before you get your Hidden Service up and running, you will need to install Tor and some other packages via OpenBSDs [package manager](https://www.openbsd.org/faq/faq15.html):

```
pkg_add -i tor torsocks vim exfat-fuse htop
```

Now, before doing anything else, save a copy of the default `torrc` config file so that we can have that back-pocket in case of any issues:

```
mkdir Documents && cp /etc/tor/torrc home/user/Documents/torrc.backup
```

## "Alright, what can I do with this thing?

Lots! I will highlight some of the things that might be worth doing, but I'm sure you can also find many others:

### Serving up a site with HTTPD
```
onion=$(cat /var/tor/hidden_service/hostname)
mkdir /var/www/htdocs/$onion
```

```
/var/www/htdocs/{}.onion/
```
### Serving up files on that site!
You can plug in an external drive to your pi and symlink the files from their to the aformentioned `/var/www/htdocs/...` to allow people to download them from tor!

Like so:

```console
ln -s /path/to/file/photo-of-my-dog.jpg /var/www/htdocs/{}.onion/photo-of-my-dog.jpg
```

Now you can send over the onion link for someone to grab via tor:

```
{}.onion/photo-of-my-dog.jpg
```

### using the server as a unix ssh box over tor

One of the many great things about a tor hidden service is that it accomplishes NAT traversal automatically through the tor network, allowing you to connect to the host here from ther road, and even other resources on the network it is being hosted within from that point. WHo needs a VPN when you have a tor hidden service running on your Pi!

Yes, this can be pretty slow, but its an option (and a nice one, at that)

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

Get things updated: systpatch, pkg_add -u

### READING MATERIALS:

'How do Onion Services work?' - https://community.torproject.org/onion-services/overview/
Openbsd's manual page for HTTPD.conf - https://man.openbsd.org/httpd.conf.5
