# Let's Encrypt Certificate Renewal with RFC2136

<h4 align="center">How to automate Let's Encrypt certificate renewal using Certbot DNS-01 challenge via RFC2136</h4>

<p align="center">
<a href="https://github.com/vttrj/certbot-rfc2136/issues"><img src="https://img.shields.io/badge/contributions-welcome-brightgreen.svg?style=flat"></a>
<a href="https://www.jvetter.net/posts/certbot-rfc2136/"><img src="https://img.shields.io/badge/🌐_-jvetter.net-blue" alt="jvetter.net"></a>
</a>
</p>

<p align="center">
  <a href="#features">Features</a> •
  <a href="#installation-instructions">Installation</a> •
  <a href="#-notes">Notes</a>
</p>


---
Let's Encrypt provides a DNS-01 challenge to verify domain ownership before issuing SSL certificates. The challenge requires adding TXT records to your DNS zone. 
Instead of manually adding DNS records, we use RFC2136 (DNS Dynamic Updates) with BIND9 to automate the process.

# Features

DNS Dynamic update :
- ✅ Automates renewal of wildcard (*.example.com) and apex (example.com) certificates
-  ✅ RFC standarized and supported accross several DNS software (e.g., PowerDNS, Knot DNS, NSD)

# Installation Instructions
## BIND9 configuration 
### 1-Key creation
Create a TSIG key for authentication. Run the following command to generate a TSIG key that Certbot will use for authentication: 

```sh
sudo rndc-confgen -a -A hmac-sha512 -k "certbot." -c /etc/bind/certbot.key
```
This creates `/etc/bind/certbot.key` with content:

```sh
key "certbot." {
    algorithm hmac-sha512;
    secret "SECRET_KEY_HERE";
};
```
### 2-Configure BIND9 to Allow Dynamic Updates

Edit `/etc/bind/named.conf.local` and add:

```sh
key "certbot." {
    algorithm hmac-sha512;
    secret "SECRET_KEY_HERE";
};

zone "_acme-challenge.jvetter.net" {
    type master;
    file "/var/cache/bind/_acme-challenge.jvetter.net.zone";
    allow-query { any; };
    check-names ignore; // required for `_acme-challenge`
    update-policy {
        grant certbot. name _acme-challenge.jvetter.net. txt;
    };
};
```
Verify you have this file taking into account in the main named.conf
In the file `/etc/bind/named.conf` verify the presence of

```sh
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
```

### 3-Create the DNS Zone File
Create the file `/var/cache/bind/_acme-challenge.jvetter.net.zone`:

```sh
$ORIGIN .
$TTL 3600  ; 1 hour
_acme-challenge.jvetter.net IN SOA ns1.jvetter.net. hostmaster.jvetter.net. (
                                44         ; serial
                                14400      ; refresh (4 hours)
                                3600       ; retry (1 hour)
                                604800     ; expire (1 week)
                                3600       ; minimum (1 hour)
                                )
                        NS      ns1.jvetter.net.
```

### 4-Verify BIND Configuration
Run:

```sh
sudo named-checkconf   # Verify named.conf syntax
sudo named-checkzone _acme-challenge.jvetter.net /var/cache/bind/_acme-challenge.jvetter.net.zone

```

Reload BIND:
```sh
sudo systemctl reload named
```

Confirm zone visibility:
```sh
dig SOA _acme-challenge.jvetter.net @ns1.jvetter.net +short
```

### 5-Test DNS Updates Using `nsupdate`

Let's test if DNS Dynamic Update for the new zone is accepted.

On the server that is running certbot (it can be where Bind is present or another server, in my case Bind and Certbot are in two different server), you need:
- The TSIG key previously created
- `nsupdate` installed (part of the package `bind9-dnsutils`)

Verify if you have `nsupdate`:
```sh
$ which nsupdate
/usr/bin/nsupdate
```
If not present, install it with:
```sh
$ sudo apt install bind9-dnsutils
```

Now run the following command:
```sh
echo -e "server ns1.jvetter.net\nzone _acme-challenge.jvetter.net\nupdate add _acme-challenge.jvetter.net 300 TXT \"test1\"\nsend\nquit" | sudo nsupdate -k certbot.key

```

Confirm the update:
```sh
dig TXT _acme-challenge.jvetter.net @ns1.jvetter.net +short
```

The following command delete only test1 TXT record:

```sh
$ echo -e "server ns1.jvetter.net\nzone _acme-challenge.jvetter.net\nupdate delete _acme-challenge.jvetter.net. IN TXT \"test1\"\nsend\nquit" | sudo nsupdate -v -d -k /etc/letsencrypt/certbot.key
```

The following command delete all TXT records:

```sh
$ echo -e "server ns1.jvetter.net\nzone _acme-challenge.jvetter.net\nupdate delete _acme-challenge.jvetter.net TXT\nsend\nquit" | sudo nsupdate -v -d -k /etc/letsencrypt/certbot.key
```

Now we confirmed the certbot server can push DNS Dynamic Update (RFC2136) to the server that manage the `_acme-challenge` zone, let’s continue with the certbot configuration.

## Certbot Configuration with RFC2136
### 6-Install the Certbot RFC2136 Plugin

On the Certbot server:
```sh
sudo apt install python3-certbot-dns-rfc2136
```

Verify:
```sh
$ certbot plugins

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
* dns-rfc2136
Description: Obtain certificates using a DNS TXT record (if you are using BIND
for DNS).
Interfaces: Authenticator, Plugin
Entry point: dns-rfc2136 =
certbot_dns_rfc2136._internal.dns_rfc2136:Authenticator

* standalone
Description: Spin up a temporary webserver
Interfaces: Authenticator, Plugin
Entry point: standalone = certbot._internal.plugins.standalone:Authenticator

* webroot
Description: Place files in webroot directory
Interfaces: Authenticator, Plugin
Entry point: webroot = certbot._internal.plugins.webroot:Authenticator
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

### 7-Configure Certbot to Use RFC2136
Create a credentials file:
```sh
sudo vi /etc/letsencrypt/renewal/rfc2136-credentials.ini
```

Add:
```sh
# Target DNS server (IP address of the DNS server)
dns_rfc2136_server = 141.94.168.204
# Target DNS port
dns_rfc2136_port = 53
# TSIG key name (matches BIND config)
dns_rfc2136_name = certbot.
# TSIG key secret (from certbot.key)
dns_rfc2136_secret = "SECRET_KEY_HERE"
# TSIG key algorithm
dns_rfc2136_algorithm = HMAC-SHA512
```

Set permissions:
```sh
sudo chmod 600 /etc/letsencrypt/renewal/rfc2136-credentials.ini
```

Certbot also needs information about how it will request a certificate. 
Edit the `your_domain.conf` file in the `letsencrypt/renewal` directory:
```sh
$ sudo cat /etc/letsencrypt/renewal/jvetter.net.conf
# renew_before_expiry = 30 days
version = 2.1.0
archive_dir = /etc/letsencrypt/archive/jvetter.net
cert = /etc/letsencrypt/live/jvetter.net/cert.pem
privkey = /etc/letsencrypt/live/jvetter.net/privkey.pem
chain = /etc/letsencrypt/live/jvetter.net/chain.pem
fullchain = /etc/letsencrypt/live/jvetter.net/fullchain.pem

# Options used in the renewal process
[renewalparams]
account = 0123456789abcdef0123456789abcdef
authenticator = dns-rfc2136
dns_rfc2136_credentials = /etc/letsencrypt/renewal/rfc2136-credentials.ini
server = https://acme-v02.api.letsencrypt.org/directory
key_type = ecdsa
```

### 8-Request SSL Certificates Using RFC2136

First test with `dry-run` if you would be able to request a certificate:

```bash
$ sudo certbot certonly --dns-rfc2136 --dns-rfc2136-credentials /etc/letsencrypt/renewal/rfc2136-credentials.ini -d "jvetter.net" -d "*.jvetter.net" --dry-run
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Simulating renewal of an existing certificate for jvetter.net and *.jvetter.net
Waiting 60 seconds for DNS changes to propagate
The dry run was successful.
```


Now you can request a certificate, run the previsous command without `dry-run`:

```sh
$ sudo certbot certonly --dns-rfc2136 --dns-rfc2136-credentials /etc/letsencrypt/renewal/rfc2136-credentials.ini -d "jvetter.net" -d "*.jvetter.net"
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Renewing an existing certificate for jvetter.net and *.jvetter.net
Waiting 60 seconds for DNS changes to propagate

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/jvetter.net/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/jvetter.net/privkey.pem
This certificate expires on 2025-05-19.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

## Automating Certificate Renewal
When using Certbot, no manual intervention is needed for certificate renewal. Certbot automatically installs a scheduled task that checks and renews certificates twice a day — at 00:00 and 12:00 UTC.

Certbot sets up for you two things to make sure the renwal is scheduled:
- a cronjob task
- a systemd.timer

Check the cron job:
```sh
$ sudo cat /etc/cron.d/certbot

# /etc/cron.d/certbot: crontab entries for the certbot package
#
# Upstream recommends attempting renewal twice a day
#
# Eventually, this will be an opportunity to validate certificates
# haven't been revoked, etc.  Renewal will only occur if expiration
# is within 30 days.
#
# Important Note!  This cronjob will NOT be executed if you are
# running systemd as your init system.  If you are running systemd,
# the cronjob.timer function takes precedence over this cronjob.  For
# more details, see the systemd.timer manpage, or use systemctl show
# certbot.timer.
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(43200))' && certbot -q renew --no-random-sleep-on-renew
```
☝️ The condition `! -d /run/systemd/system` ensures the cron job is skipped if systemd is managing the service, avoiding duplicate renewals.

The systemd timer unit (preferred when systemd is present):
```sh
$ sudo systemctl cat certbot.timer

# /lib/systemd/system/certbot.timer
[Unit]
Description=Run certbot twice daily

[Timer]
OnCalendar=*-*-* 00,12:00:00
RandomizedDelaySec=43200
Persistent=true

[Install]
WantedBy=timers.target
```

## Reloading Nginx Docker After Renewal
After renewing an SSL certificate, Nginx must be reloaded to use the new certificate. However, when Nginx is running inside a Docker container, Certbot doesn’t automatically reload it.
To handle this, we’ll configure a systemd override for the certbot.service to include a `--post-hook` that restarts the Nginx container after a successful renewal.

### Create a systemd override for certbot service unit:
```sh
sudo mkdir -p /etc/systemd/system/certbot.service.d
```

Create the override file:
```sh
sudo vi /etc/systemd/system/certbot.service.d/override.conf
```

Add the following content:
```sh
[Service]
ExecStart=
ExecStart=/usr/bin/certbot -q renew --post-hook "/usr/bin/docker-compose -f /home/debian/nginx-redirect/docker-compose.yml restart nginx"
``` 
### Reload systemd and restart the timer:
```sh
sudo systemctl daemon-reload
sudo systemctl restart certbot.timer
```

### Verify the override config is applied:
The base unit and the override should be visible:
```sh
$ sudo systemctl cat certbot.service

# /lib/systemd/system/certbot.service
[Unit]
Description=Certbot
Documentation=file:///usr/share/doc/python-certbot-doc/html/index.html
Documentation=https://certbot.eff.org/docs
[Service]
Type=oneshot
ExecStart=/usr/bin/certbot -q renew --no-random-sleep-on-renew
PrivateTmp=true

# /etc/systemd/system/certbot.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/bin/certbot -q renew --post-hook "/usr/bin/docker-compose -f /home/debian/nginx-redirect/docker-compose.yml restart nginx"
```

☝️ The `--post-hook` only runs after a successful renewal.

# Verify Renewal and Deployment
After the renewal process, you should verify both:

1. That Certbot successfully renewed the certificate
2. That Nginx is now serving the updated certificate

## Check Certificate Status with Certbot
This shows the certificates managed by Certbot:
```sh
$ sudo certbot certificates

Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Found the following certs:
  Certificate Name: jvetter.net
    Serial Number: 386c024c9cc53219a2512be3b026f94e46f
    Key Type: ECDSA
    Domains: jvetter.net *.jvetter.net
    Expiry Date: 2025-05-20 21:53:59+00:00 (VALID: 85 days)
    Certificate Path: /etc/letsencrypt/live/jvetter.net/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/jvetter.net/privkey.pem
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```
☝️ Certbot displays the certificate it created, not what is actively served by Nginx.

## Check the File Directly Using OpenSSL
You can verify the dates on the actual certificate file that Certbot generated:
```sh
$ sudo openssl x509 -noout -dates -in /etc/letsencrypt/live/jvetter.net/fullchain.pem
notBefore=Feb 19 21:54:00 2025 GMT
notAfter=May 20 21:53:59 2025 GMT
```

## Verify What Nginx is Actually Serving
To ensure Nginx is using the renewed certificate, connect to the server directly over TLS:
```sh
$ echo | openssl s_client -connect jvetter.net:443 -servername jvetter.net 2>/dev/null | openssl x509 -noout -dates

notBefore=Feb 19 21:54:00 2025 GMT
notAfter=May 20 21:53:59 2025 GMT
```
☝️ These dates should match the ones displayed by Certbot and the certificate file.

## Use Online SSL Checkers
You can also use website dedicated for that like :

- [https://www.ssllabs.com](https://www.ssllabs.com/)
- https://www.sslshopper.com/ssl-checker.html
- [https://web-check.xyz](https://web-check.xyz/) (my favorite for its UI ♥️)



# 📋 Notes

- Certbot version 2.1.0 (on Debian 12 Bookworm)
- DNS server runs BIND version 9.16.50 (on Debian 11 Bullseye)
