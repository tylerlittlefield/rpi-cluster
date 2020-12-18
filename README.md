
<!-- README.md is generated from README.Rmd. Please edit that file -->

# rpi-cluster

<!-- badges: start -->
<!-- badges: end -->

Notes on my Raspberry Pi cluster ❤️

## Hardware

-   MacBook Pro 2019 16" laptop to work with the Pi
-   3 Raspberry Pi’s, 8GB model
-   1 MicroSD card
-   3 USB &gt;= 3.0
-   1 Network switch with 4 PoE ports
-   3 PoE hats
-   3 CAT6 cables
-   1 Cluster case that has 3 layers

## Setting up the microSD card

First step is to configure a microSD card that’s running Raspberry Pi
OS. I wont go into detail on how that part is done, check the official
imaging tools, it’s a simple process. Additionally, **I am not using
WiFi at all**, so I skip steps like setting up the `wpa_supplicant.conf`
file.

Anyway, after you’ve written the Pi OS to a microSD card enable SSH:

``` bash
touch /Volumes/boot/ssh
```

Now unmount the drive, insert it into the Pi and turn it on! If all goes
as planned, you can find your Pi’s IP with `nmap`:

``` bash
nmap -sn 192.168.1.0/24 # ip may very, check what your ip is
```

You should see something like “raspberrypi”, identify the IP associated
with this and SSH into the device. From here, we can update the EEPROM:

``` bash
# pw: raspberry
ssh pi@<IP found in above step>

# upgrade eeprom and reboot
sudo apt-get upgrade -y
sudo rpi-eeprom-update -a
sudo reboot
```

**This next step may not be applicable to others** but this was the key
to get my Pi’s booting from USB (I think, trial and error makes it all
this fuzzy). We need to modify the boot order of the Pi’s, I discovered
this bit of information from:

-   <https://jamesachambers.com/new-raspberry-pi-4-bootloader-usb-network-boot-guide/>

SSH back in the Pi and run:

``` bash
# verify that the BOOT_ORDER=0x1 line is changed to BOOT_ORDER=0x4
sudo -E rpi-eeprom-config --edit

# then reboot
sudo reboot
```

SSH back in the Pi and shut it down for now:

``` bash
sudo shutdown -h now
```

We are ready to set up the USB boot!

## Setting up USB boot

Just as the imager tool was used with the microSD, do the same but with
Ubuntu Server 20.10 on a USB device. Once this completes, we need to
edit some files, I discovered this information from:

-   <https://github.com/DavidUnboxed/Ubuntu-20.04-WiFi-RaspberyPi4B>

We need to edit:

1.  `user-data`
2.  `network-config`

You can reference the repository above, or just reference the code
below. For `network-config` we need something like:

``` bash
version: 2
ethernets:
  eth0:
    # Rename the built-in ethernet device to "eth0"
    match:
      driver: bcmgenet smsc95xx lan78xx
    set-name: eth0
    dhcp4: true
    optional: true
```

For `user-data` we need something like:

``` bash
#cloud-config

chpasswd:
  expire: true
  list:
  - ubuntu:ubuntu

ssh_pwauth: true

power_state:
  mode: reboot
```

If I remember correctly, the `power_state` part was needed to get WiFi
working, and I ditched using WiFi entirely, so it may not be needed
anymore (assuming you are going ethernet only route).

Now, with the Pi shutdown and disconnected from power, insert the USB
and remove the microSD. Turn on the Pi and it should boot from USB now.
Repeat this process for all Pi’s.

## Changing the hostname

For all my Pi’s, I want clear hostnames to organize my cluster. I went
with the following naming:

1.  main
2.  worker-01
3.  worker-02

To do this, SSH in the Pi and run:

``` bash
sudo hostnamectl set-hostname <desired hostname>
```

Now reboot the device with `sudo reboot`. Repeat all the stuff with the
other Pi’s.

## Change user

The default user is `ubuntu` but I want to change it. In my case, I have
a user `pirate` for all Pi’s. To do this, I ran:

``` bash
# check groups of ubuntu
groups

# add a user
sudo adduser pirate

# set groups for pirate (i found these groups by running `groups` as you see above)
sudo usermod -a -G adm,dialout,cdrom,floppy,sudo,audio,dip,video,plugdev,netdev,lxd pirate
```

Log out of the Pi with `exit` and try to log in with your new username.
If you can login with out issue, delete the `ubuntu` user:

``` bash
sudo deluser --remove-home ubuntu
```

## Set up `.pub` key authentication

I think there are multiple ways you can set up key based authentication,
there is a `.pem` and `.pub` file, I’m not sure which is better or what
the difference is but I’m using `.pub`. This allows your to SSH into the
Pi’s without entering passwords and has a number of other benefits like
allowing Ansible to control your Pi cluster. To do this, I start by
making a key if you don’t already have one:

``` bash
ssh-keygen -t rsa -b 4096 -C "tyler@rpi" # put whatever comment you want
```

Now copy the key to each Pi:

``` bash
ssh-copy-id -i ~/.ssh/id_rsa.pub pirate@<ip>
```

Next time you SSH, no password should be required.

## Setting up k3s

tbc

## Setting up monitoring tools

tbc