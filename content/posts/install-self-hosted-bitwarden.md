---
title: "Install self-hosted Bitwarden"
date: 2021-07-14T22:17:24-05:00
draft: true
---

Instructions at https://bitwarden.com/help/article/install-on-premise/

Create a virtual machine to host the instance
---------------------------------------------

Create a new volume in the pool for the VM. Use the Debian generic cloud image
as a backing file.

```
hypervisor$ sudo virsh vol-create-as lab bitwarden 10G --format qcow2 \
  --backing-vol debian-10-generic-amd64.qcow2 --backing-vol-format qcow2
```

Create the iso to inject configuration in cloud-init.

```
hypervisor$ make_cidata_iso bitwarden
hypervisor$ CIDATA=cidata-bitwarden.iso
hypervisor$ size=$(stat -c%s $CIDATA)
hypervisor$ sudo virsh vol-create-as lab $CIDATA $size
hypervisor$ sudo virsh vol-upload $CIDATA $CIDATA --pool lab
```

Generate a MAC address and add it to the virtual network:

```
hypervisor$ $ sudo virsh net-update default add-last ip-dhcp-host \
  --xml "<host mac='52:54:00:00:00:02' ip='192.168.122.2' name='bitwarden' />" \
  --live --config
```

Install the guest domain:

```
hypervisor$ sudo virt-install --name bitwarden --vcpus 2 --ram 2048 --hvm \
  --disk vol=lab/bitwarden --disk vol=lab/cidata-bitwarden.iso,device=cdrom \
  --os-type=linux --os-variant=debian10 --boot hd --noautoconsole --autostart \
  -w bridge=virbr0,model=virtio,mac=52:54:00:00:00:02
```

Forward incoming connections. See intructions at
https://wiki.libvirt.org/page/Networking#Forwarding_Incoming_Connections.

Don't forget to restart the guest (stop/start, not reboot).

```
hypervisor$ sudo virsh shutdown bitwarden
hypervisor$ sudo virsh start bitwarden
```

We can connect to the guest domain:

```
hypervisor$ ssh 192.168.122.2
```

Prepare the environment
-----------------------

Install docker and docker-compose:

```
bitwarden:~$ sudo apt update
bitwarden:~$ sudo apt install docker-compose
```

Create `bitwarden` user:

```
bitwarden:~$ sudo adduser --system --home /opt/bitwarden --ingroup docker bitwarden
bitwarden:~$ sudo chmod -R 700 /opt/bitwarden/
```

Install Bitwarden
-----------------

Open a shell as `bitwarden` user:

```
bitwarden:~$ sudo /bin/su - bitwarden -s /bin/bash
bitwarden@bitwarden:~$ pwd
/opt/bitwarden
```

Review the installer script.

Download the installation script in the home and set execution permission.

```
$ curl -LsO https://raw.githubusercontent.com/btravouillon/server/dns-gandi/scripts/bitwarden.sh \
    && chmod 700 bitwarden.sh
```

Setup the configuration file with the LiveDNS API key:

```
$ mkdir -p /opt/bitwarden/bwdata/letsencrypt
$ echo "dns_gandi_api_key=$GANDI_LIVEDNS_KEY" > /opt/bitwarden/bwdata/letsencrypt/gandi.ini
$ chmod 600 /opt/bitwarden/bwdata/letsencrypt/gandi.ini
```

Execute the installer:
```
bitwarden@bitwarden:~$ ./bitwarden.sh install
 _     _ _                         _
| |__ (_) |___      ____ _ _ __ __| | ___ _ __
| '_ \| | __\ \ /\ / / _` | '__/ _` |/ _ \ '_ \
| |_) | | |_ \ V  V / (_| | | | (_| |  __/ | | |
|_.__/|_|\__| \_/\_/ \__,_|_|  \__,_|\___|_| |_|

Open source password management solutions
Copyright 2015-2021, 8bit Solutions LLC
https://bitwarden.com, https://github.com/bitwarden

===================================================

bitwarden.sh version 1.41.3
Docker version 20.10.5+dfsg1, build 55c4c88
docker-compose version 1.25.0, build unknown

(!) Enter the domain name for your Bitwarden instance (ex. bitwarden.example.com): vault.example.com

(!) Do you want to use Let's Encrypt to generate a free SSL certificate? (y/n): y

(!) Enter your email address (Let's Encrypt will send you certificate expiration reminders): john@example.com

Using default tag: latest
latest: Pulling from certbot/certbot
Digest: sha256:1de29b86aa08f09b944b89dc74f1b3a4789b53eb7addeb2b29c276ec730a402f
Status: Image is up to date for certbot/certbot:latest
docker.io/certbot/certbot:latest
Sending build context to Docker daemon  41.96MB
Step 1/2 : FROM certbot/certbot
 ---> da5b74422e31
Step 2/2 : RUN pip install certbot-plugin-gandi
 ---> Using cache
 ---> 5ce296f9ca48
Successfully built 5ce296f9ca48
Successfully tagged certbot/dns-gandi:latest
Saving debug log to /etc/letsencrypt/logs/letsencrypt.log
[...]
Account registered.
Requesting a certificate for vault.example.com
Waiting 10 seconds for DNS changes to propagate

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/vault.example.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/vault.example.com/privkey.pem
This certificate expires on 2021-09-25.
These files will be updated when the certificate renews.

NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1.41.3: Pulling from bitwarden/setup
69692152171a: Pull complete
6142a500eec7: Pull complete
98e06df84f9c: Pull complete
ebbd8772c8dd: Pull complete
9c620bc3b6b2: Pull complete
c2c1c2b5cec1: Pull complete
b1bdb6c4f7eb: Pull complete
1d942d7b2d5a: Pull complete
e0acb3bad2b1: Pull complete
db47512b40e0: Pull complete
Digest: sha256:71d8647e0e4e5d9494188ef59e67850208aafabf8ae6cb49cc9ff60493d6e3db
Status: Downloaded newer image for bitwarden/setup:1.41.3
docker.io/bitwarden/setup:1.41.3

(!) Enter your installation id (get at https://bitwarden.com/host): <use your installation id>

(!) Enter your installation key: <use your installation key>

Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time

[...]
Generating key for IdentityServer.
Generating a RSA private key
writing new private key to 'identity.key'
-----

Building nginx config.
Building docker environment files.
Building docker environment override files.
Building FIDO U2F app id.
Building docker-compose.yml.

Installation complete

If you need to make additional configuration changes, you can modify
the settings in `./bwdata/config.yml` and then run:
`./bitwarden.sh rebuild` or `./bitwarden.sh update`

Next steps, run:
`./bitwarden.sh start`

```

Start bitwarden:
```
bitwarden@bitwarden:~$ ./bitwarden.sh update
 _     _ _                         _
| |__ (_) |___      ____ _ _ __ __| | ___ _ __
| '_ \| | __\ \ /\ / / _` | '__/ _` |/ _ \ '_ \
| |_) | | |_ \ V  V / (_| | | | (_| |  __/ | | |
|_.__/|_|\__| \_/\_/ \__,_|_|  \__,_|\___|_| |_|

Open source password management solutions
Copyright 2015-2021, 8bit Solutions LLC
https://bitwarden.com, https://github.com/bitwarden

===================================================

bitwarden.sh version 1.41.3
Docker version 20.10.5+dfsg1, build 55c4c88
docker-compose version 1.25.0, build unknown

"docker inspect" requires at least 1 argument.
See 'docker inspect --help'.

Usage:  docker inspect [OPTIONS] NAME|ID [NAME|ID...]

Return low-level information on Docker objects
1.41.3: Pulling from bitwarden/setup
Digest: sha256:71d8647e0e4e5d9494188ef59e67850208aafabf8ae6cb49cc9ff60493d6e3db
Status: Image is up to date for bitwarden/setup:1.41.3
docker.io/bitwarden/setup:1.41.3

Building docker environment files.
Building docker environment override files.
Building nginx config.
Building FIDO U2F app id.
Building docker-compose.yml.

Pulling mssql         ... done
Pulling web           ... done
Pulling attachments   ... done
Pulling api           ... done
Pulling identity      ... done
Pulling sso           ... done
Pulling admin         ... done
Pulling portal        ... done
Pulling icons         ... done
Pulling notifications ... done
Pulling events        ... done
Pulling nginx         ... done
Using default tag: latest
latest: Pulling from certbot/certbot
Digest: sha256:1de29b86aa08f09b944b89dc74f1b3a4789b53eb7addeb2b29c276ec730a402f
Status: Image is up to date for certbot/certbot:latest
docker.io/certbot/certbot:latest
Sending build context to Docker daemon  42.15MB
Step 1/2 : FROM certbot/certbot
 ---> da5b74422e31
Step 2/2 : RUN pip install certbot-plugin-gandi
 ---> Using cache
 ---> 5ce296f9ca48
Successfully built 5ce296f9ca48
Successfully tagged certbot/dns-gandi:latest

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Processing /etc/letsencrypt/renewal/vault.example.com.conf
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Certificate not yet due for renewal

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
The following certificates are not due for renewal yet:
  /etc/letsencrypt/live/vault.example.com/fullchain.pem expires on 2021-09-25 (skipped)
No renewals were attempted.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Saving debug log to /etc/letsencrypt/logs/letsencrypt.log
Creating directory /opt/bitwarden/bwdata/core
Creating directory /opt/bitwarden/bwdata/core/attachments
Creating directory /opt/bitwarden/bwdata/logs
Creating directory /opt/bitwarden/bwdata/logs/admin
Creating directory /opt/bitwarden/bwdata/logs/api
Creating directory /opt/bitwarden/bwdata/logs/events
Creating directory /opt/bitwarden/bwdata/logs/icons
Creating directory /opt/bitwarden/bwdata/logs/identity
Creating directory /opt/bitwarden/bwdata/logs/mssql
Creating directory /opt/bitwarden/bwdata/logs/nginx
Creating directory /opt/bitwarden/bwdata/logs/notifications
Creating directory /opt/bitwarden/bwdata/logs/sso
Creating directory /opt/bitwarden/bwdata/logs/portal
Creating directory /opt/bitwarden/bwdata/mssql/backups
Creating directory /opt/bitwarden/bwdata/mssql/data
Creating network "docker_default" with the default driver
Creating network "docker_public" with the default driver
Creating bitwarden-attachments   ... done
Creating bitwarden-web           ... done
Creating bitwarden-icons         ... done
Creating bitwarden-mssql         ... done
Creating bitwarden-sso           ... done
Creating bitwarden-api           ... done
Creating bitwarden-notifications ... done
Creating bitwarden-events        ... done
Creating bitwarden-identity      ... done
Creating bitwarden-admin         ... done
Creating bitwarden-portal        ... done
Creating bitwarden-nginx         ... done
1.41.3: Pulling from bitwarden/setup
Digest: sha256:71d8647e0e4e5d9494188ef59e67850208aafabf8ae6cb49cc9ff60493d6e3db
Status: Image is up to date for bitwarden/setup:1.41.3
docker.io/bitwarden/setup:1.41.3


Bitwarden is up and running!
===================================================

visit https://vault.example.com
to update, run `./bitwarden.sh updateself` and then `./bitwarden.sh update`

Total reclaimed space: 0B
Pausing 60 seconds for database to come online. Please wait...
1.41.3: Pulling from bitwarden/setup
Digest: sha256:71d8647e0e4e5d9494188ef59e67850208aafabf8ae6cb49cc9ff60493d6e3db
Status: Image is up to date for bitwarden/setup:1.41.3
docker.io/bitwarden/setup:1.41.3

Migrating database.
Migration successful.
Database update complete
```

Copy the certs
```
bitwarden@bitwarden:~$ scp -r bwdata/letsencrypt/live/vault.example.com/ proxy:/etc/ssl/
```

Edit nginx.conf on reverse proxy:
```
    server {
        listen         443;
        server_name    vault.example.com;

        ssl                  on;
        ssl_certificate      /etc/ssl/vault.example.com/fullchain.pem;
        ssl_certificate_key  /etc/ssl/vault.example.com/privkey.pem;

        ### SSL log files ###
        access_log      /var/log/nginx/ssl-access.log;
        error_log       /var/log/nginx/ssl-error.log;

        location / {
            proxy_pass   https://192.168.10.25:3443;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_redirect https://192.168.10.25:3443 https://vault.example.com;
        }
    }
```

Restart:
```
# /etc/init.d/nginx restart
```


Update bwdata/env/global.override.env with globalSettings__mail__smtp*:

Backup: rsync, rush, etc

Restore from scratch:
curl script, restore /opt/bitwarden/bwdata, run ./bitwarden.sh update

```
$ pwd
/opt/bitwarden
$ whoami 
bitwarden
$ curl -LsO https://raw.githubusercontent.com/btravouillon/server/dns-gandi/scripts/bitwarden.sh \
    && chmod 700 bitwarden.sh
$ tar xf bitwarden-backup.tar.xz
$ ./bitwarden.sh update
```
