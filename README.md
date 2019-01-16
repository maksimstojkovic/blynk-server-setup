# Blynk Local Server Setup

- [Blynk Local Server Setup](#blynk-local-server-setup)
  - [Initial Setup](#initial-setup)
  - [Server Properties](#server-properties)
  - [Mail Properties](#mail-properties)
  - [Generating SSL Certificates](#generating-ssl-certificates)
  - [Firewall Settings](#firewall-settings)
  - [Router Settings](#router-settings)
  - [Useful Links](#useful-links)

## Initial Setup
Install Java 8 JDK
```
sudo apt-get install oracle-java8-jdk
```

Ensure Java 8 was installed
```
java -version
```

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
https.port=19443
http.port=18080
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
admin.rootPath=/homeiot-blynk-login
allowed.administrator.ips=192.168.1.0/24
admin.email=<SERVER_ADMIN_EMAIL_HERE>
admin.pass=<SERVER_ADMIN_PASS_HERE>
contact.email=<SERVER_ADMIN_EMAIL_HERE>
server.host=<HOST_ADDRESS_HERE>
```
**Note:** `https.port` and `http.port` can be set to any port, but firewall and router settings must match the ports used.

**Note:** `allowed.administrator.ips` means that accessing the server admin page can only be done using the local ID address of the server (e.g. https://192.168.1.x:19443/homeiot-blynk-login)

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

Make both scripts executable
```
chmod +x auth.sh cleanup.sh
```

Run `certbot` in manual mode to generate an SSL certificate for the server. The DNS-01 challenge should be used during this process, along with the scripts that were just created.
```
sudo certbot certonly --manual --preferred-challenges dns --manual-auth-hook /path/to/auth.sh --manual-cleanup-hook /path/to/cleanup.sh
```

**Note:** The paths to `auth.sh` and `cleanup.sh` should be absolute paths.

When prompted, enter the full DuckDNS domain name (e.g. `domain.duckdns.org`). Completing this procedure should yield a certificate and private key in the directory: `/etc/letsencrypt/live/domain.duckdns.org/`. Ensure that the paths to `fullchain.pem` and `privkey.pem` are consistent with the paths in the `server.properties` file.


## Firewall Settings
Add `ufw` rules which allow for incoming TCP requests on the HTTP (18080) and HTTPS (19443) ports.
```
sudo ufw allow 18080/tcp comment 'Blynk HTTP'
sudo ufw allow 19443/tcp comment 'Blynk HTTPS'
```

## Router Settings
Port forward the HTTP (18080) and HTTPS (19443) ports on your router to point towards the server's local IP address.

For assistance check [Port Forward][7].

## Useful Links
1. [blynk-server Repository][1]
2. [Blynk Documentation][2]
3. [Let's Encrypt Forum - DuckDNS Verification][3] - See post by _jmorahan_ on 01/02/18
4. [Port Fortward][4]



<!-- References -->
[1]: https://github.com/blynkkk/blynk-server
[2]: https://docs.blynk.cc/
[3]: https://community.letsencrypt.org/t/raspberry-pi-with-duckdns-ddns-failing-to-verify/53567/8
[4]: https://portforward.com/