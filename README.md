# Secure DNS with your Raspberry (how to install Pi-Hole with DNS over HTTPS)

## High Level Design - System Architecture
The system is based on a *Pi-Hole* server which is offering the **DNS** functionality which is using *Cloudflared* to hide DNS over HTTPS connecting to *Mullvad*.

The system will accept requests **DNS over TLS** thanks to *Stubby* that will redirect the requests to *Pi-Hole* and *Pi-Hole* will send these requests to *Mullvad* using *Cloudflared* and **DNS over HTTP**.

* Stubby
* Pihole
* Cloudflared

```mermaid
flowchart  LR

A[DNS Request DNS over TTL]  -->  B[DNS Server - Raspberry Pi]
B  -->  C[Stubby]
C  -->  D[Pi-hole]
D  -->  E[Cloudflared]
E  -->  F[Mullvad DNS Server]
F  -->  G[Internet]
```

## Steps

### Step 1 (configure your SD Card to bool your Raspberry)

Let's start with a fresh image installed with Raspberry Pi Imager, as I am using a *Pi Zero 2 W* so I will install *Raspberry Pi OS Lite 64 bits*.

Remember to configure your WiFi settigns before recording the image. Finish this step putting the image on the Pi.

### Step 2 (update everything)
Start your PI with the flashed card.

Update everything, SO, packages and Firmware:
> sudo apt update && sudo apt full-upgrade -y && sudo rpi-update
> sudo reboot

### Step 3 (install Pi-Hole)
Let's install the main package, *Pi-Hole*
>curl -sSL https://install.pi-hole.net | bash

Select the following wizard options:
* Continue on the static IP (ensure you have a static IP, I do it with a DHCP reservation but it is up to you).
* Upstream DNS: select Custom
* Enter the following IP address (we will change all this afterwards): 194.242.2.2
* Enable query loging to allow for better statistics recording everything (this is all optional, you can use your criteria here).

You should have now the pihole server installed. Configure the password for your webserver doing the following:
>sudo pihole setpassword *yourpassword*

Access its own URL **ipaddress/admin** and navigate to the *Lists* section of the menu. There add a new list to block:
>https://big.oisd.nl
>https://energized.pro/ultimate/hosts.txt

After doing this update the gravity with:
>sudo pihole -g



