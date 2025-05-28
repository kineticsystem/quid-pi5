# Raspian OS Lite and Squid on Pi 5 <!-- omit in toc -->

This guide is to install Squid on a Rasperry PI 5 with an M.2 SSD and Raspian OS Lite.

## Table of contents <!-- omit in toc -->

- [Notes](#notes)
- [Hardware prerequisites](#hardware-prerequisites)
- [Configuration](#configuration)
- [Remote installation](#remote-installation)
- [Prepare the Bootable Drive](#prepare-the-bootable-drive)
- [Boot the PI](#boot-the-pi)
  - [Create the Raspian OS Lite image](#create-the-raspian-os-lite-image)
  - [SSH connection](#ssh-connection)
- [Install Squid](#install-squid)
- [Squid configuration](#squid-configuration)
  - [Generate the certificates](#generate-the-certificates)
  - [Prepare SSL certificate database](#prepare-ssl-certificate-database)
- [Create and prepare the cache folder](#create-and-prepare-the-cache-folder)
- [Update Squid configuration](#update-squid-configuration)
- [Initialize and start](#initialize-and-start)
  - [Quick client configuration](#quick-client-configuration)
  - [Safe client configuration](#safe-client-configuration)
- [Test](#test)
- [Docker containers](#docker-containers)
- [External links](#external-links)

## Notes

With Ubuntu 24.04LTS, the Rasperry Pi 5 is too slow, ssh is not working well and the board runs very hot.

## Hardware prerequisites

This is a suggested configuration for good performance and reliability.

- A Rasperry PI 5 8Gb RAM
- A Raspberry pi 5 power supply
- An Active Cooler compatible with Raspberry Pi 5
- An M.2 SSD 2T
- Geekworm X1001 PCIe to M.2 NVMe SSD PIP PCIe Peripheral Board for Raspberry Pi
- Geekworm Raspberry Pi 5 Case

## Configuration

host: **squid.local**

user: **pi**

password: **#########**

## Remote installation

With the latest firmare installed, if you connect the Pi to the router and boot it up with an empty SD Card, USB key or M.2 SSD, it will downbload Imager from internet, execute it in RAM and allow you to install any OS.

See here: https://www.youtube.com/watch?v=prqzOg-GC40

This would be the best option, if the firmware is already up-to-date. Otherwise, to install Raspian Lite on your PI 5 with 2TB M.2 SSD, follow these steps.

## Prepare the Bootable Drive

You can boot the Rasperry Pi 5 with a USB key with **Raspbian OS Desktop**. At this stage, we need the desktop version because we want to update the Pi 5 firmware and install a software called Imager.

To create the bootable USB with Raspbian OS Desktop, use the Imager application on your normal PC. You can install Imager with the following command:

```bash
sudo apt install rpi-image
```

## Boot the PI

Insert the USB with Raspian OS Desktop and boot the Pi 5. You do not need the M.2 SSD installed at this point, because we only want to update the OS and the firmware.

Once the OS starts, open a terminal an type:

```bash
sudo apt update && sudo apt full-upgrade
```

Reboot, open a terminal and type:

```bash
sudo rpi-eeprom-update
```

If the firmware needs update, type

```bash
sudo rpi-eeprom-update -a
```

At this point you can shutdown the Pi 5, and install the SSD if you haven't done already.

### Create the Raspian OS Lite image

Boot up the Raspberry Pi 5 using the USB key with Raspian OS Desktop and install Imager. Use it to create the Raspian Lite image on the M.2 SSD. Remember to enable WiFi and SSH connection.

Remove the USB and reboot.

### SSH connection

If you can ping the new device, but you cannot log in using SSH, you may need to run the following command to enable it.

```bash
sudo raspi-config
```

## Install Squid

First, update the APT configuration file `etc/apt/sources.list`:

```bash
sudo vim etc/apt/sources.list
```

Uncomment all sources:

```bash
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian-security/ bookworm-security main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian-security/ bookworm-security main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

Update the package database:

```bash
sudo apt update
```

Install the required dependencies to build Squid.

```bash
sudo apt install devscripts
sudo apt build-dep squid
```

Install the Squid source package, without using `sudo`:

```bash
apt source squid
cd squid-5.7
```

Compile Squid. The command will generate two builds, one with SSL support and one without. The building process takes around 30 minutes.

```bash
debuild -us -uc
```

Once completed install the package with SSL support.

```bash
sudo dpkg -i ../squid-openssl_5.7-2+deb12u2_arm64.deb ../squid-common_5.7-2+deb12u2_all.deb
```

If there are dependency issues, fix them with:

```bash
sudo apt install -f
```

To verify if SSL support is enabled, run this:

```bash
/usr/sbin/squid -v | grep -i ssl
```

You should see SSL/TLS related features listed.

```bash
This binary uses OpenSSL 3.0.16 - OpenSSL library is linked and ready
--with-openssl - OpenSSL support enabled during compilation
--enable-ssl-crtd - SSL certificate daemon enabled (crucial for SSL bump/HTTPS
```

What this means:

- Your Squid can now handle HTTPS traffic
- It can perform SSL bumping (decrypt/inspect HTTPS traffic)
- It can act as an SSL proxy
- It supports certificate generation for intercepted HTTPS connections

Execute Squid:

```bash
sudo systemctl start squid
sudo systemctl status squid
```

## Squid configuration

### Generate the certificates

```bash
# Create the certificates folder.
sudo mkdir -p /etc/squid/cert
cd /etc/squid/cert

# Generate CA private key.
sudo openssl genrsa -out ca.key 2048

# Generate CA certificate (valid for one year).
sudo openssl req -x509 -new -nodes -sha256 -days 365 -key ca.key -out ca.pem -subj '/C=PL/ST=Poland/O=MyOrg/CN=Self-signed CA'

# Add DER format, useful for some browsers.
sudo openssl x509 -in ca.pem -outform DER -out ca.der

# Generates dhparam.pem for stronger cryptography.
sudo openssl dhparam -outform PEM -out dhparam.pem 2048

# Set ownership and permissions.
sudo chown -R proxy:proxy /etc/squid/cert
sudo chmod 700 /etc/squid/cert
```

The certificate generated at this step will be used to re-encrypt your HTTPS traffic, `ca.pem` file must be imported to your clients web-browser.

### Prepare SSL certificate database

Create the folder where we will store the fake SSL certificates for HTTPS interception.

```bash
sudo /usr/lib/squid/security_file_certgen -c -s /var/spool/squid/ssl_db -M 4MB
sudo chown -R proxy:proxy /var/spool/squid/ssl_db/
```

## Create and prepare the cache folder

Create the folder where we will store the cache.

```bash
sudo mkdir -p /var/cache/squid
sudo chown proxy:proxy /var/cache/squid
```

## Update Squid configuration

Backup Squid configurationn file.

```bash
sudo cp /etc/squid/squid.conf /etc/squid/squid.conf.bak
```

Replace the existing configuration with this:

```bash
# Allow certificate fetching during SSL handshake.
acl intermediate_fetching transaction_initiator certificate-fetching
http_access allow intermediate_fetching

# SSL certificate generator settings.
sslcrtd_program /usr/lib/squid/security_file_certgen -s /var/spool/squid/ssl_db -M 4MB
sslcrtd_children 8 startup=1 idle=1

# SSL bumping method - "Stare" at SSL traffic first, then decide what to do.
ssl_bump stare all

# Your 1.5TB cache directory
cache_dir ufs /var/cache/squid 1500000 16 256

# SSL bumping port - a lot of rules.
http_port 3129 ssl-bump generate-host-certificates=on dynamic_cert_mem_cache_size=4MB cert=/etc/squid/cert/ca.pem key=/etc/squid/cert/ca.key cipher=HIGH:MEDIUM:!LOW:!RC4:!SEED:!IDEA:!3DES:!MD5:!EXP:!PSK:!DSS options=NO_TLSv1,NO_SSLv3 tls-dh=prime256v1:/etc/squid/cert/dhparam.pem

# Network access control (adjust to your network).
acl localnet src 192.168.0.0/16  # Define: "localnet" = any IP from 192.168.x.x
http_access allow localnet       # Rule 1: ALLOW traffic from localnet
http_access deny all             # Rule 2: DENY everything else

# Allow cache management from localhost.
acl manager proto cache_object
acl localhost src 127.0.0.1/32 ::1
http_access allow manager localhost
```

## Initialize and start

Initialize the cache directory.

```bash
sudo systemctl stop squid
sudo squid -z
```

Start Squid.

```bash
sudo systemctl start squid
sudo systemctl enable squid
```

### Quick client configuration

If you do not want the client to validate the self-signed certificate, you can add the following into your `~/.bashrc` file. Otherwise skip to the next section.

```bash
export http_proxy=http://squid.local:3129
export https_proxy=http://squid.local:3129
export HTTPS_PROXY=http://squid.local:3129
export HTTP_PROXY=http://squid.local:3129
export PYTHONHTTPSVERIFY=0
export CURL_CA_BUNDLE=""
export REQUESTS_CA_BUNDLE=""
export SSL_VERIFY=false
```

### Safe client configuration

On the client, add the following to your `~/.bashrc` file.

```bash
export http_proxy="http://squid.local:3129"
export https_proxy="http://squid.local:3129"
export HTTP_PROXY="http://squid.local:3129"
export HTTPS_PROXY="http://squid.local:3129"
export no_proxy="localhost,127.0.0.1,:1"
```

On the PI run the following commands:

```bash
# Copy the certificate to a location you can access
sudo cp /etc/squid/cert/ca.pem ~/squid-ca.crt
sudo chown pi:pi ~/squid-ca.crt
```

On Ubuntu run the following commands:

```bash
# Copy from Pi to Ubuntu (replace PI_IP with your Pi's IP)
scp pi@squid.local:~/squid-ca.crt ~/
```

Install the CA certificate system-wide:

```bash
# Copy to system certificate directory
sudo cp ~/squid-ca.crt /usr/local/share/ca-certificates/

# Update system certificates
sudo update-ca-certificates

# Verify it was added
ls /usr/local/share/ca-certificates/
```

To configure APT to use proxy, create the following file.

```bash
sudo gedit /etc/apt/apt.conf.d/95proxies
```

Add the following content.

```bash
Acquire::http::Proxy "http://squid.local:3129";
Acquire::https::Proxy "http://squid.local:3129";
```

## Test

```bash
# Test HTTP
curl -v http://example.com

# Test HTTPS
curl -v https://example.com

# Check if traffic is going through proxy
sudo tail -f /var/log/squid/access.log  # (run this on the Pi)
```

## Docker containers

To make the proxy work in the docker container, you can choose to ingnore the self signed certificate validation.

Here is an example on how to attach to a running container:

```bash
docker compose exec -it \
    -e http_proxy=http://192.168.100.25:3129 \
    -e https_proxy=http://192.168.100.25:3129 \
    -e HTTPS_PROXY=http://192.168.100.25:3129 \
    -e HTTP_PROXY=http://192.168.100.25:3129 \
    -e PYTHONHTTPSVERIFY=0 \
    -e CURL_CA_BUNDLE="" \
    -e REQUESTS_CA_BUNDLE="" \
    -e SSL_VERIFY=false \
    dev bash
```

## External links

https://gist.github.com/avoidik/a2b3762bad03931755d5dd190cbd0112
