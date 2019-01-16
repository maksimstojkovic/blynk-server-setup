# Blynk Local Server Setup
This document outlines the procedures for installing a local Blynk server on a Raspberry Pi (Zero) with SSL certificates. At the time of writing this the Blynk local server version is: `0.41.2`.

Make sure you understand the terms and conditions of services used to replicate the instructions below.

**Warning:** This information may become outdated very quickly.

## Table of Contents
- [Blynk Local Server Setup](#blynk-local-server-setup)
  - [Table of Contents](#table-of-contents)
  - [Pre-Requisites](#pre-requisites)
  - [Initial Setup](#initial-setup)
  - [Server Properties](#server-properties)
  - [Mail Properties](#mail-properties)
  - [Firewall Settings](#firewall-settings)
  - [Generating SSL Certificates](#generating-ssl-certificates)
  - [Accept the IP logging notice.](#accept-the-ip-logging-notice)
  - [Are you OK with your IP being logged?](#are-you-ok-with-your-ip-being-logged)
  - [Renewing SSL Certificates](#renewing-ssl-certificates)
  - [Running the Server](#running-the-server)
  - [Starting Server on Reboot](#starting-server-on-reboot)
  - [Updating the Server](#updating-the-server)
  - [Router Settings](#router-settings)
  - [Sketches With (HTTP) and Without (HTTPS) SSL](#sketches-with-http-and-without-https-ssl)
  - [Useful Links](#useful-links)

## Pre-Requisites
You will need a Raspberry Pi device which meets the specifications outlined [here](https://github.com/blynkkk/blynk-server#requirements). A Raspberry Pi Zero W running Raspbian Stretch and an ESP32 Devkit V1 module were used to test the instructions outlined in this document.

Before proceeding, register a DuckDNS account and install the appropriate cronjob on your server using the documentation available [here](https://www.duckdns.org/install.jsp). Make sure you refer to instructions for the correct operating system. Note that DuckDNS cannot be accessed when connected to a VPN.

Additionally, the Raspberry Pi used for testing had `ufw` installed to 'uncomplicate' the firewall aspect of running a server.

## Initial Setup
Install Java 8 JDK.
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install oracle-java8-jdk
```

Ensure Java 8 is installed.
```
java -version
```
**Note:** Java 8 does not have to be updated to the latest release (particularly relevant to Raspberry Pi as standard apt-get commands may not necessary retrieve the latest Java version).

Create a new folder named `blynk` in the current user's home directory.
```
mkdir blynk
```

Download the Blynk Server jar in the new `blynk` directory.
```
cd blynk
wget "https://github.com/blynkkk/blynk-server/releases/download/v0.41.2/server-0.41.2-java8.jar"
```
**Note:** The download link is subject to change. Please refer to [blynk-server Repository](https://github.com/blynkkk/blynk-server#quick-local-server-setup-on-raspberry-pi).

## Server Properties
Create a new file called `server.properties` in the same directory as the server jar.
```
nano server.properties
```

Insert the following settings:
```
https.port=9443
http.port=8080
data.folder=/home/<USERNAME_HERE>/blynk/data
logs.folder=/home/<USERNAME_HERE>/blynk/logs
log.level=trace
server.ssl.cert=/etc/letsencrypt/live/<DUCKDNS_ADDRESS_HERE>/fullchain.pem
server.ssl.key=/etc/letsencrypt/live/<DUCKDNS_ADDRESS_HERE>/privkey.pem
server.ssl.key.pass=
user.dashboard.max.limit=5
user.message.quota.limit=100
notifications.frequency.user.quota.limit=60
user.profile.max.size=128
terminal.strings.pool.size=25
notifications.queue.limit=5000
blocking.processor.thread.pool.limit=6
profile.save.worker.period=60000
hard.socket.idle.timeout=15
enable.raw.data.store=false
admin.rootPath=/blynk-login
allowed.administrator.ips=192.168.1.0/24
admin.email=<SERVER_ADMIN_EMAIL_HERE>
admin.pass=<SERVER_ADMIN_PASS_HERE>
contact.email=<SERVER_ADMIN_EMAIL_HERE>
server.host=<DUCKDNS_ADDRESS_HERE>
```
**Note:** `https.port` and `http.port` can be set to any port, but firewall and router settings must match the ports used. The `ssl` property formatting <span style="text-decoration:underline">MUST</span> be followed closely, otherwise SSL connections will not work. The property `server.ssl.key.pass` <span style="text-decoration:underline">MUST</span> be kept in the file with an empty value.

**Note:** `allowed.administrator.ips` means that accessing the server admin page can only be done within the local network by using the local IP address of the server (e.g. https://192.168.1.40:9443/blynk-login). CIDR notation can be used to indicate an IP range (try entering `192.168.1.0/24` in the *CIDR to IP Range* field [here]). If the server is not accessible within the local network (e.g. a server run offsite), you may want to change this to: `0.0.0.0/0`

Additional information about the properties listed can be found [here](https://github.com/blynkkk/blynk-server#advanced-local-server-setup).

## Mail Properties
Create a new file called `mail.properties` in the same directory as the server jar.
```
nano mail.properties
```

Insert the following settings:
```
mail.smtp.auth=true
mail.smtp.starttls.enable=true
mail.smtp.host=smtp.gmail.com
mail.smtp.port=587
mail.smtp.username=<SERVER_ADMIN_EMAIL_HERE>
mail.smtp.password=<SERVER_ADMIN_EMAIL_PASS_HERE>
```

## Firewall Settings
Add `ufw` rules which allow for incoming TCP requests on the HTTP (8080) and HTTPS (9443) ports.
```
sudo ufw allow 8080/tcp comment 'Blynk HTTP'
sudo ufw allow 9443/tcp comment 'Blynk HTTPS'
```
**Note:** Only the IPv4 protocol is required for the local Blynk server ports.

**Note:** When creating a Blynk local server behind a router, it is not necessary to port forward traffic within the server itself. Instead, traffic should be port forwarded from the router to the server. A brief description is provided [here](#router-settings).

## Generating SSL Certificates
Install `certbot`.
```
sudo apt-get install certbot
```

Create a new `ssl` directory in the same directory as the blynk server jar.
```
mkdir ssl
cd ssl
```

Inside the `ssl` directory, create a script called `auth.sh`:
```
nano auth.sh
```
```
#!/bin/bash
DUCKDNS_TOKEN="<DUCKDNS_TOKEN_HERE>"
[[ "$(curl -s "https://www.duckdns.org/update?domains=${CERTBOT_DOMAIN%.duckdns.org}&token=${DUCKDNS_TOKEN}&txt=${CERTBOT_VALIDATION}")" = "OK" ]]
```

Inside the `ssl` directory, create another script called `cleanup.sh`:
```
nano cleanup.sh
```
```
#!/bin/bash
DUCKDNS_TOKEN="<DUCKDNS_TOKEN_HERE>"
[[ "$(curl -s "https://www.duckdns.org/update?domains=${CERTBOT_DOMAIN%.duckdns.org}&token=${DUCKDNS_TOKEN}&txt=${CERTBOT_VALIDATION}&clear=true")" = "OK" ]]
```

**Important:** The only change that should be made to these scripts is replacing `<DUCKDNS_TOKEN_HERE>` with the token for your DuckDNS account

Make both scripts executable.
```
chmod +x auth.sh cleanup.sh
```

Run `certbot` in manual mode to generate an SSL certificate for the server. The DNS-01 challenge should be used during this process, along with the scripts that were just created.
```
sudo certbot certonly --manual --preferred-challenges dns --manual-auth-hook <ABSOLUTE_PATH>/blynk/ssl/auth.sh --manual-cleanup-hook <ABSOLUTE_PATH>/blynk/ssl/cleanup.sh
```

**Note:** The paths to `auth.sh` and `cleanup.sh` should be absolute paths. These files must remain within the `/ssl` directory indefinitely for generation and renewal to be successful.

When prompted, enter an email address (e.g. the one used for server admin).
```
Enter email address (used for urgent renewal and security notices)
(Enter 'c' to cancel): <SERVER_ADMIN_EMAIL_HERE>
```

Accept the Terms of Service notice.
```
-------------------------------------------------------------------------------
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v01.api.letsencrypt.org/directory
-------------------------------------------------------------------------------
(A)gree/(C)ancel: A
```

When prompted, enter your full DuckDNS address (e.g. `domain.duckdns.org`).
```
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c' to cancel): domain.duckdns.org
```

Accept the IP logging notice.
-------------------------------------------------------------------------------
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
-------------------------------------------------------------------------------
(Y)es/(N)o: Y

**Important:** Make sure you read the notes produced during the creation of SSL certificates.

Completing this procedure will yield a certificate and private key in the directory: `/etc/letsencrypt/live/domain.duckdns.org/`. Ensure that the paths to `fullchain.pem` (cert) and `privkey.pem` (key) are consistent with the paths in the [`server.properties`](#server-properties) file.

## Renewing SSL Certificates
Certificate renewal can be tested using:
```
sudo certbot renew --manual-public-ip-logging-ok --dry-run
```

Assuming there are no errors, SSL certificate renewal can be automated by modifying the `certbot` crontab entry in `/etc/cron.d`.
```
sudo nano /etc/cron.d/certbot
```

Initially, the contents of this file should be:
```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(3600))' && certbot -q renew
```

Change the last line to:
```
0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(3600))' && certbot -q --manual-public-ip-logging-ok renew
```
**Note:** Certbot randomises the renewal time for certificates to distribute Let's Encrypt server loads.

## Running the Server
In order for the server to access the SSL certificates generated, the server should be run as root. In the `blynk` directory, execute the following command:
```
sudo java -jar server-0.41.2-java8.jar
```
**Important:** Make sure that the name to the server file is correct!

Although you will receive `INFO` notices stating `Didn't find Let's Encrypt certificates` and `Automatic certificate generation is turned ON`, the certificates will have been installed correctly as there are no `WARN` notices.

After running the server once until no further notices are printed (`Ctrl+C` to interrupt), check the logs in `blynk/logs/blynk.log` the make sure no errors were encountered.

## Starting Server on Reboot
As mentioned above, the server must be run with root privileges to access the SSL certificates generated. Thus, to restart the server on boot, a root cronjob is required.
```
sudo crontab -e
```
Insert the following line at the bottom of the file:
```
@reboot java -jar /home/<USERNAME_HERE>/blynk/server-0.41.2-java8.jar &
```
**Important:** *Make sure that the path and name of the server file are correct!*

**Note:** The ampersand (&) runs the task in the background, suppressing output.

After rebooting, use the following command to check whether the process is being run by root:
```
ps -aux | grep java
```

## Updating the Server
Refer to the relevant section from the blynk-server repository documentation [here](https://github.com/blynkkk/blynk-server#update-instruction-for-unix-like-systems).

## Router Settings
Port forward the HTTP (8080) and HTTPS (9443) ports on your router to the respective HTTP and HTTPS ports of your Blynk local server. Your Raspberry Pi should have a static IP address which the router can point to (learn more about setting static IPs [here][8]).

For assistance with port forwarding, check [Port Forward][7].

## Sketches With (HTTP) and Without (HTTPS) SSL
To use a SSL hardware connection (HTTPS), ensure that a SSL enabled sketch is being used (e.g. `ESP32_WiFi_SSL.ino`). These sketches include an SSL enabled Blynk library, such as:
```
#include <BlynkSimpleEsp32_SSL.h>
```

Conversely, for hardware connections without SSL (HTTP), use an appropriate sketch (e.g. `ESP32_WiFi.ino`). These sketches include a corresponding Blynk library without SSL, such as:
```
#include <BlynkSimpleEsp32.h>
```

Initialise the variables declared in the example:
* Auth Token
* SSID
* Password

Additionally, declare two more variables as shown below:
```
char server[] = "<SERVER_ADDRESS_HERE>";
int port = <PORT_HERE>;
```
For HTTPS connections, the value of server[] should be the same as the DuckDNS address used for the `server.host` property in `server.properties` ([here](#server-properties)). For HTTP connections, use either the server's local network IP address (e.g. `192.168.1.40`) for local connections, or the DuckDNS address. Use the appropriate port value (9443 for HTTPS, and 8080 for HTTP).

**Important:** You <span style="text-decoration:underline">MUST</span> use the DuckDNS address as the server[] value when using SSL. If you do not, the SSL connection will be rejected, giving the following error:
```
Connecting to 192.168.1.40:9443
Secure connection failed
```

For SSL connections, the following definition is required to change the root certificate authority used by hardware. This line should be placed at the top of SSL sketches along with other global variables:
```
#define BLYNK_SSL_USE_LETSENCRYPT
```
For more information, check the [blynk-library Repository][2]. In particular, review the structure of sketches which use SSL, such as [BlynkSimpleEsp32_SSL.h][6]

If you continue encountering errors, verify whether your device is capable of using SSL connections.

**Note:** This has only been tested using the `ESP32_WiFi_SSL.ino` example sketch after following the above server configurations.

## Useful Links
1. [blynk-server Repository][1]
2. [blynk-library Repository][2]
3. [Blynk Documentation][3]
4. [Blynk Forums][7]
5. [DuckDNS][9]
6. [Let's Encrypt Forum | DuckDNS Verification][4] - See post by _jmorahan_ on 01/02/18
7. [Setting Up a Static IP on Raspberry Pi][8]
8. [Port Fortward][5]

<!-- References -->
[1]: https://github.com/blynkkk/blynk-server
[2]: https://github.com/blynkkk/blynk-library
[3]: https://docs.blynk.cc/
[4]: https://community.letsencrypt.org/t/raspberry-pi-with-duckdns-ddns-failing-to-verify/53567/8
[5]: https://portforward.com/
[6]: https://github.com/blynkkk/blynk-library/blob/master/src/BlynkSimpleEsp32_SSL.h
[7]: https://community.blynk.cc/
[8]: https://www.raspberrypi.org/learning/networking-lessons/rpi-static-ip-address/
[9]: https://www.duckdns.org/