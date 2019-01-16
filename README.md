# Blynk Server Setup
- [Initial Setup](#initial-setup)
- [Server Properties](#server-properties)
- [Mail Properties](#mail-properties)
- [Generating SSL Certificates](#generating-ssl-certificates)
- [Firewall Settings](#firewall-settings)
- [Router Settings](#router-settings)

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
server.ssl.cert=/etc/letsencrypt/live/<HOST_ADDRESS_HERE>/fullchain.pem
server.ssl.key=/etc/letsencrypt/live/<HOST_ADDRESS_HERE>/privkey.pem
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

Inside the `ssl` directory, create two new shell scripts: `auth.sh` and `cleanup.sh`.

## Firewall Settings
Add `ufw` rules which allow for incoming TCP requests on the HTTP (18080) and HTTPS (19443) ports.
```
sudo ufw allow 18080/tcp comment 'Blynk HTTP'
sudo ufw allow 19443/tcp comment 'Blynk HTTPS'
```

## Router Settings
Port forward the HTTP (18080) and HTTPS (19443) ports on your router to point towards the server's local IP address.

For assistance check [Port Forward][2].




<!-- References -->
[1]: https://github.com/blynkkk/blynk-server
[2]: https://portforward.com/