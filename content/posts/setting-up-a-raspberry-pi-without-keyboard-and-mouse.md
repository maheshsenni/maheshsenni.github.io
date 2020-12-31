---
title: "Setting Up a Raspberry Pi Without Keyboard and Mouse (Headless)"
date: 2020-01-29T13:54:27-06:00
draft: false
---

If you are someone like me without a USB keyboard, mouse or a LAN cable, it is not straight forward to install [Raspbian](https://www.raspbian.org/) on a [Raspberry Pi 3](https://www.raspberrypi.org/). As the latest releases of Raspbian come with [SSH disabled](https://www.raspberrypi.org/blog/a-security-update-for-raspbian-pixel/) by default for security reasons, it took me a while to figure out a way to set it up with SSH enabled so that I can install and configure other tools on the Pi.

Here are two methods that have worked for me.

- Using **NOOBS** to install Raspbian
- Burn Raspbian image to SD card (I prefer this)

If you have a USB keyboard and mouse, using NOOBS gives you an interactive interface to easily install Raspbian (or any other OS). But it is challenging to use NOOBS if you don’t have the USB devices. I find method 2 to be the most straight forward to setup Raspbian with SSH and Wi-Fi connectivity preconfigured. Once you are able to SSH into the Raspberry Pi, you should be able to accomplish most of the other configurations.

### Required Tools

- Raspberry Pi 3 model B
- A computer that you can use to prepare the SD card and SSH into the Pi
- Wi-Fi network

Wi-Fi network is not required if you have a LAN cable that you can use to connect the Raspberry Pi and router.

### Method 1

1. Download NOOBS from here — [https://www.raspberrypi.org/downloads/noobs/](https://www.raspberrypi.org/downloads/noobs/)

2. Format the SD using a tool like [SD Memory Card Formatter](https://www.sdcard.org/downloads/formatter_4/index.html) if it is not done already.

3. Extract and copy contents to SD card

4. Assuming that you want to install Raspbian, delete every other operating system directory from `os` directory except for `Raspbian`

5. Append `silentinstall` to recovery.cmdline file the in root directory

6. Edit /os/Raspbian/partition_setup.sh in the SD card and add the following lines after the three `sed` commands after replacing the [two letter ISO code](https://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) for your country, Wi-Fi network name and password. If you have a LAN cable to connect your Pi and router, you can remove the lines after and including `#Configure WiFi`.

7. Eject the SD card and insert it into the Raspberry Pi

8. If you have got an HDMI cable, connect the Pi to a display as it really helps to monitor the progress of the installation.

9. Power up the Pi and wait for the installation to complete

10. From your laptop, use a tool like [Nmap](https://nmap.org/) to scan your local network. If you don’t have **Nmap**, look at the steps [here](https://superuser.com/a/1009536) to install it. I have used `192.168.0.1/24` for the CIDR range to scan but this depends on your local network. Find out your machine’s IP address in the local network and use that in the CIDR range.

11. **Nmap** scan should provide a list of IP addresses corresponding to the devices connected to your network. Usually, the last IP address in the scan will be Pi’s as it is the latest device to join the network.

12. Once you know the Pi’s IP address, SSH into it as the default `pi` user with the password `raspberry`. It is best to change to change the default password once you login for security.

### Method 2

This method installs an image of Raspbian directly in the SD card using an image writing tool like [Etcher](https://etcher.io/).

1. Download the latest image of Raspbian from here — [https://www.raspberrypi.org/downloads/raspbian/](https://www.raspberrypi.org/downloads/raspbian/)

2. Format the SD using a tool like [SD Memory Card Formatter](https://www.sdcard.org/downloads/formatter_4/index.html) if it is not done already.

3. I am using **Etcher** for writing the image to the SD card. Download the latest version of Etcher from here — [https://etcher.io/](https://etcher.io/)

4. Open Etcher and select the image downloaded earlier. Ensure that you have selected the correct SD card and flash it.

5. Once the flash is complete, you should see a **boot** volume created in your SD card.

6. For enabling SSH in the Pi, create a file named SSH in the root of the boot volume.

7. To preconfigure the Pi to connect to a WiFi network, create a file called wpa_supplicant.conf in the root of the **boot** volume with the following contents. This is not required if you have a LAN cable to connect the Pi to your router. You can skip this step if you have a LAN cable to connect your Pi and router.

8. Eject the SD card and insert it into the Raspberry Pi

9. If you have got an HDMI cable, connect the Pi to a display as it helps to find out any problems when the Pi boots for the first time.

10. Power up the Pi and give it a minute to boot

11. From your laptop, use a tool like [Nmap](https://nmap.org/) to scan your local network. If you don’t have Nmap, look at the steps [here](https://superuser.com/a/1009536) to install it. I have used `192.168.0.1/24` for the CIDR range to scan but this depends on your local network. Find out your machine’s IP adress in the local network and use that in the CIDR range.

12. Nmap scan should provide a list of IP addresses corresponding to the devices connected to your network. Usually, the last IP address in the scan will be Pi’s as it is the latest device to join the network.

13. Once you know the Pi’s IP address, SSH into it as `pi` user with the default password `raspberry`. It is best to change to change the default password once you login for security.

If you have come this far, you should be able to make changes to your Raspberry Pi using SSH. Have a ☕️ and enjoy your Pi.

> This post was originally [published in Medium](https://medium.com/@maheshsenni/setting-up-a-raspberry-pi-without-keyboard-and-mouse-headless-9359e0926807) on Feb 10, 2018

### References

[https://raspberrypi.stackexchange.com/a/31594](https://raspberrypi.stackexchange.com/a/31594)

[https://github.com/raspberrypi/noobs/blob/master/README.md](https://github.com/raspberrypi/noobs/blob/master/README.md)

[https://www.raspberrypi.org/forums/viewtopic.php?f=63&t=141559](https://www.raspberrypi.org/forums/viewtopic.php?f=63&t=141559)

[https://github.com/raspberrypi/noobs/issues/365](https://github.com/raspberrypi/noobs/issues/365)

[https://raspberrypi.stackexchange.com/questions/15192/installing-raspbian-from-noobs-without-display](https://raspberrypi.stackexchange.com/questions/15192/installing-raspbian-from-noobs-without-display)

[http://blog.smalleycreative.com/linux/setup-a-headless-raspberry-pi-with-raspbian-jessie-on-os-x/](http://blog.smalleycreative.com/linux/setup-a-headless-raspberry-pi-with-raspbian-jessie-on-os-x/)
