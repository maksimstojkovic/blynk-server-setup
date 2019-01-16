# Blynk Local Server Setup
This document outlines the procedures for installing a local Blynk server on a Raspberry Pi (Zero) with SSL certificates.

# Table of Contents
- [Blynk Local Server Setup](#blynk-local-server-setup)
- [Table of Contents](#table-of-contents)
  - [Pre-Requisites](#pre-requisites)
  - [Initial Setup](#initial-setup)
  - [Server Properties](#server-properties)
  - [Mail Properties](#mail-properties)
  - [Firewall Settings](#firewall-settings)
  - [Generating SSL Certificates](#generating-ssl-certificates)
  - [Renewing SSL Certificates](#renewing-ssl-certificates)
  - [Running the Server](#running-the-server)
  - [Starting Server on Reboot](#starting-server-on-reboot)
  - [Updating the Server](#updating-the-server)
  - [Router Settings](#router-settings)
  - [Sketches With (HTTP) and Without (HTTPS) SSL](#sketches-with-http-and-without-https-ssl)
  - [Useful Links](#useful-links)

## Pre-Requisites
You will need a Raspberry Pi device which meets the specifications outlined [here](https://github.com/blynkkk/blynk-server#requirements). A Raspberry Pi Zero W running Raspbian Stretch and an ESP32 Devkit V1 module were used to test the instructions outlined in this document.

Before proceeding, register a DuckDNS account and install it on your server using the documentation available [here](https://www.duckdns.org/install.jsp). Make sure you refer to instructions for the correct operating system.

## Initial Setup
Install Java 8 JDK
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install oracle-java8-jdk
```

Ensure Java 8 was installed.
```
java -version
```
**Note:** Java 8 does not have to be updated to the latest release (particularly relevant to Raspberry Pi as standard apt-get commands may not necessary retrieve the latest version).

Create a new folder named `blynk` in the current user's home directory
```
mkdir blynk
```

Download the Blynk Server jar in the new `blynk` directory
```
cd blynk
wget "https://github.com/blynkkk/blynk-server/releases/download/v0.41.2/server-0.41.2-java8.jar"
```
**Note:** This link is subject to change. Refer to [blynk-server Repository][1].

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
**Note:** `https.port` and `http.port` can be set to any port, but firewall and router settings must match the ports used.

Additional information about the properties listed can be found [here]https://github.com/blynkkk/blynk-server#advanced-local-server-setup).

**Note:** `allowed.administrator.ips` means that accessing the server admin page can only be done within the local network by using the local IP address of the server (e.g. https://192.168.1.x:9443/blynk-login). If the server is not accessible within the local network, you may want to change this to: `0.0.0.0/0`

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

## Generating SSL Certificates
Install `certbot`.
```
sudo apt-get install certbot
```

Create a new `ssl` directory in the same directory as the blynk server jar.
```
mkdir ssl
```

Inside the `ssl` directory, create a script called `auth.sh`:
```
#!/bin/bash
DUCKDNS_TOKEN="<DUCKDNS_TOKEN_HERE>"
[[ "$(curl -s "https://www.duckdns.org/update?domains=${CERTBOT_DOMAIN%.duckdns.org}&token=${DUCKDNS_TOKEN}&txt=${CERTBOT_VALIDATION}")" = "OK" ]]
```

Inside the `ssl` directory, create another script called `cleanup.sh`:
```
#!/bin/bash
DUCKDNS_TOKEN="<DUCKDNS_TOKEN_HERE>"
[[ "$(curl -s "https://www.duckdns.org/update?domains=${CERTBOT_DOMAIN%.duckdns.org}&token=${DUCKDNS_TOKEN}&txt=${CERTBOT_VALIDATION}&clear=true")" = "OK" ]]
```

Make both scripts executable
```
chmod +x auth.sh cleanup.sh
```

Run `certbot` in manual mode to generate an SSL certificate for the server. The DNS-01 challenge should be used during this process, along with the scripts that were just created.
```
sudo certbot certonly --manual --preferred-challenges dns --manual-auth-hook <ABSOLUTE_PATH>/blynk/ssl/auth.sh --manual-cleanup-hook <ABSOLUTE_PATH>/blynk/ssl/path/to/cleanup.sh
```

**Note:** The paths to `auth.sh` and `cleanup.sh` should be absolute paths. These files must remain within the `/ssl` directory for renewal to be successful.

When prompted, enter the full DuckDNS domain name (e.g. `domain.duckdns.org`). Completing this procedure should yield a certificate and private key in the directory: `/etc/letsencrypt/live/domain.duckdns.org/`. Ensure that the paths to `fullchain.pem` and `privkey.pem` are consistent with the paths in the `server.properties` file.

## Renewing SSL Certificates
Testing of certificate renewal can be performed using:
```
sudo certbot renew --manual-public-ip-logging-ok --dry-run
```

Assuming there are no errors, SSL certificate renewal can be automated by modifying the `certbot` crontab entry in `/etc/cron.d`. Initially, the contents of this file should be:
```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(3600))' && certbot -q
```

Change the last line to:
```
0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(3600))' && certbot -q --manual-public-ip-logging-ok renew
```
**Note:** Certbot randomises the renewal time for certificates to distribute Let's Encrypt server loads.

## Running the Server
In order for the server to access the SSL certificates generated, the server should be run as root. In the `blynk` directory, execute the following command:
```
java -jar server-0.41.2-java8.jar
```
**Important:** *Make sure that the name to the server file is correct!*

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

## Updating the Server
Refer to the relevant section from the blynk-server repository documentation [here](https://github.com/blynkkk/blynk-server#update-instruction-for-unix-like-systems).

## Router Settings
Port forward the HTTP (8080) and HTTPS (9443) ports on your router to point towards the server's local IP address.

For assistance check [Port Forward][7].

## Sketches With (HTTP) and Without (HTTPS) SSL
To use a SSL hardware connection (HTTPS), ensure that a SSL enabled sketch is being used (e.g. `ESP32_WiFi_SSL.ino`). These sketches include an SSL enabled Blynk library, such as:
```
#include <BlynkSimpleEsp32_SSL.h>
```

Conversely, for hardware connections without SSL (HTTP), use an appropriate sketch (e.g. `ESP32_WiFi.ino`). These sketches include a WiFi Blynk library without SSL, such as:
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

For SSL connections, the following definition is required to change the root certificate authority used by hardware. This definition should be placed at the top of sketch along with other global variables:
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
5. [Let's Encrypt Forum | DuckDNS Verification][4] - See post by _jmorahan_ on 01/02/18
6. [Port Fortward][5]

<!-- References -->
[1]: https://github.com/blynkkk/blynk-server
[2]: https://github.com/blynkkk/blynk-library
[3]: https://docs.blynk.cc/
[4]: https://community.letsencrypt.org/t/raspberry-pi-with-duckdns-ddns-failing-to-verify/53567/8
[5]: https://portforward.com/
[6]: https://github.com/blynkkk/blynk-library/blob/master/src/BlynkSimpleEsp32_SSL.h
[7]: https://community.blynk.cc/