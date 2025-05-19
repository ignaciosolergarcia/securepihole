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

```mermaid
flowchart  LR
A[DNS Request] --> B[DNS Server - Raspberry Pi]
B --> C[Pi-hole]
C --> D[Cloudflared]
D --> E[Mullvad DNS Server]
E --> F[Internet]
```
## Network design
This tutorial is designed to work with the following network topology that presents specific challenges and it is considered to be a common one. We will identify the steps linked to this topology so you can adapt them to your own if needed.

```mermaid

flowchart LR

A[Internet] --> B[DNS Request]

A[Internet] --> C[DNS over TTL Request]

A[Internet] --> D[External Web domain]

B --> E[Router]

C --> E[Router]

D --> E[Router]

E --> F[Internal Network]

F --> H[Pi-Hole]

G[Internal Web domain] --> H

```
We want to access all the services offered by the *Pi-Hole* from the *Internet* and the *Network* as well. In our case the Router has a dyndns functionality so we will be using that domain to access the services from internet. We will refer to that domain as the `dyndns-router-internet-domain`, to access these services from within the network we will use the `internal.dyndns-router-internet-domain`

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
Before installing the main package let's install lighthttp so we can serve the *Pi-Hole* web using it (otherwise the web will be served by the *Pi-Hole* service itself but it does not allow us to use SSL certificates.
>sudo apt install lighttpd
>sudo apt install php-cgi php-common php
>sudo systemctl restart lighttpd

Now, let's install the main package, *Pi-Hole*
>curl -sSL https://install.pi-hole.net | bash

Select the following wizard options:
* Continue on the static IP (ensure you have a static IP, I do it with a DHCP reservation but it is up to you).
* Upstream DNS: select Custom
* Enter the following IP address (we will change all this afterwards): 194.242.2.2 (it is the *Mullvad* server address)
* Enable query loging to allow for better statistics recording everything (this is all optional, you can use your criteria here).

You should have now the pihole server installed. Configure the password for your webserver doing the following:
>sudo pihole setpassword *yourpassword*

Access its own URL **ipaddress/admin** and navigate to the *Lists* section of the menu. There add these two new lists to block:
>https://big.oisd.nl

>https://energized.pro/ultimate/hosts.txt

After doing this update the gravity with:
>sudo pihole -g

### Step 4 (prepare the context to properly run  Pi-Hole)
As Pi-Hole performance can degrade over time due to multiple reasons and as we are running on low specs hardware let's ensure we get the system always as performant as today:

Create the file `/etc/pihole/pihole-FTL.conf`and setup the following values:

```  bash
MAXDBDAYS=7
MAXLOGAGE=72
```
This will ensure that the system performs overtime but will prevent us from having statistics for longer than a week. You can  tweak the `MAXDBDAYS` value upon your specific needs.

Another thing to protect your system is to ensure that  no log file grows unlimited generating issues with free space. To do it create a file called `/etc/logrotate.d/custom-logs` with the following content.

``` bash
/var/log/*.log  {
size 10M
rotate  5
compress
missingok
notifempty
create  0640  root  adm
}
```
Last but not least, given the hardware being used restarting the server weekly will help to keep the performance stable. I suggest doing it with cron like this:
>sudo crontab -e
>0 4 * * 0 systemctl restart pihole-FTL

That's all regarding performance of the system, now let's put the right certificate to ensure that we can access the website using SSL (this is optional but I don't like navigating with HTTP at all, even at home).

Install *certbot* in order to generate the certificate:
>sudo apt install certbot

And then generate the certificate like this (of course you will need a domain name):
>sudo systemctl stop pihole-FTL
>sudo certbot certonly --standalone -d `dyndns-router-internet-domain`
>sudo systemctl start pihole-FTL

### Step 5 (install Cloudflared)
