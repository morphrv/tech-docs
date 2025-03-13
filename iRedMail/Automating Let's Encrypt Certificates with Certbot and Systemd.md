# Automating Let's Encrypt Certificates with Certbot and Systemd

## Introduction

This guide explains how to automate Let's Encrypt certificate renewals for iRedMail using Certbot's systemd timer instead of the cron-based approach in iRedMail's official docs. While Certbot's snap package defaults to a timer, integrating it with iRedMail's services (Dovecot, Postfix, Nginx) requires custom steps - especially for updating certificates and restarting services. This fills a documentation gap for snap-based setups relying on the systemd timer service and will help you achieve fully automated certificates from Let's Encrypt without cron.

## Prerequisites

- iRedMail installed and running successfully as per existing [installation instructions](https://docs.iredmail.org/index.html)
- Certbot installed via snap as per the Certbot documentation (e.g. [Nginx on Snap](https://certbot.eff.org/instructions?ws=nginx&os=snap))
- An existing domain with access for DNS management (e.g. Cloudflare for wildcard certificates)
- Root or sudo access to your server
- Basic familiarity with systemd and shell scripting

### Background

Certbot's snap package uses a systemd timer (`snap.certbot.renew.timer`) to check for renewals twice daily, unlike the cron jobs documented in older guides. This guide adapts that timer for iRedMail by adding a post-renewal hook to update certificates and restart services.

## Initial Setup (Optional)

### Requesting a Certificate with Webroot

- Install the `certbot` package first
  - Certbot recommends using the `snapd` package which is installed by default on Ubuntu 18.04 and above. Other distributions will need to follow the [installation instructions](https://snapcraft.io/docs/installing-snapd)
- `sudo snap install --classic certbot`
- Let's Encrypt has request rate limit control. To ensure that you don't inadvertently hit that limit it is recommended that a `dry-run` is performed to verify the command and all other requirements are met before attempting to get a production certificate.

```bash
sudo certbot certonly --webroot --dry-run -w /opt/www/well_known -d mail.mydomain.com
```

- For multiple domains (SANs), append additional `-d` arguments:

```bash
sudo certbot certonly --webroot --dry-run \
 -w /opt/www/well_known \
 -d mail.domain.com \
 -d 2nd-domain.com \
 -d 3rd-domain.com
```

If successful, request a production certificate by omitting `--dry-run`:

```bash
sudo certbot certonly --webroot -w /opt/www/well_known -d mail.mydomain.com
```

Certificates are stored in `/etc/letsencrypt/live/mail.mydomain.com/`:

- `cert.pem`: Server certificate
- `chain.pem`: Additional intermediate certificate or certificates that web browsers will need in order to validate the server certificate.
- `fullchain.pem`: All certificates, including the server certificate. The server certificate is listed first in this file, followed by any intermediate certificates.
- `privkey.pem`: Private key for the certificate.

> [!NOTE]
> If port 80 isn't available, use [Certbot with a DNS plugin](https://eff-certbot.readthedocs.io/en/latest/using.html#dns-plugins) - additional setup required, not covered here.

### Using DNS Plugins (Overview)

If webroot isn't an option (e.g. existing webserver or wildcard needs), install a DNS plugin: `sudo snap install certbot-dns-<PLUGIN>`. See [full instructions](https://eff-certbot.readthedocs.io/en/latest/using.html#dns-plugins).

### Use the Let's Encrypt Certificate

Backup and symlink certificates:

- on RHEL/CentOS

```bash
sudo mv /etc/pki/tls/certs/iRedMail.crt{,.bak}       # Backup
sudo mv /etc/pki/tls/private/iRedMail.key{,.bak}     # Backup
sudo ln -s /etc/letsencrypt/live/mail.mydomain.com/fullchain.pem /etc/pki/tls/certs/iRedMail.crt
sudo ln -s /etc/letsencrypt/live/mail.mydomain.com/privkey.pem /etc/pki/tls/private/iRedMail.key
```

- on Debian/Ubuntu, FreeBSD and OpenBSD:

```bash
sudo mv /etc/ssl/certs/iRedMail.crt{,.bak}       # Backup. Rename iRedMail.crt to iRedMail.crt.bak
sudo mv /etc/ssl/private/iRedMail.key{,.bak}     # Backup. Rename iRedMail.key to iRedMail.key.bak
sudo ln -s /etc/letsencrypt/live/mail.mydomain.com/fullchain.pem /etc/ssl/certs/iRedMail.crt
sudo ln -s /etc/letsencrypt/live/mail.mydomain.com/privkey.pem /etc/ssl/private/iRedMail.key
```

> [!NOTE]
> File names are **case sensitive**

Restart services:

```bash
sudo systemctl restart postfix dovecot nginx
```

#### Verify Certificate

- Use a mail client (e.g., Thunderbird) to test Postfix/Dovecot SSL.
- For Nginx, check via browser or [SSL Labs](https://www.ssllabs.com/ssltest/index.html)
- Command-line: `openssl s_client -connect yourdomain.com:993` (Dovecot), `587` (Postfix), `443` (Nginx).

## Automate Renewals (with Certbot DNS Plugin & Systemd Timers)

### 1. Verifying Certificate Paths

Check:

```bash
sudo ls -l /etc/letsencrypt/live/yourdomain.com
```

Expected:

```bash
lrwxrwxrwx 1 root root  48 Mar 13 11:32 fullchain.pem -> ../../archive/yourdomain.com/fullchain.pem
lrwxrwxrwx 1 root root  46 Mar 13 11:32 privkey.pem -> ../../archive/yourdomain.com/privkey.pem
```

iRedMail paths:

- RHEL/CentOS: `/etc/pki/tls/certs/iRedMail.crt`,`/etc/pki/tls/private/iRedMail.key`
- Debian/Ubuntu, FreeBSD, OpenBSD: `/etc/ssl/certs/iRedMail.crt`,`/etc/ssl/private/iRedMail.key`

### 2. Create a Certbot Post-Renewal Hook

Certbot can execute a script after a successful renewal using the `--deploy-hook` option. The default directory for these hooks is `/etc/letsencrypt/renewal-hooks/deploy`.

Create the hook script to update the symlinks and restart the relevant services automatically:

```bash
sudo nano /etc/letsencrypt/renewal-hooks/deploy/iredmail-cert-update.sh
```

Add the following content:

```bash
#!/bin/bash
DOMAIN="yourdomain.com"
LIVE_DIR="/etc/letsencrypt/live/$DOMAIN"
ln -sf "$LIVE_DIR/fullchain.pem" /etc/ssl/certs/iRedMail.crt
ln -sf "$LIVE_DIR/privkey.pem" /etc/ssl/private/iRedMail.key
systemctl restart dovecot postfix nginx
```

#### Optional Enhancement to Script (Logging)

Add the following line after the `systemctl` line:

```bash
echo "$(date): Certificate renewed, services restarted" >> /var/log/iredmail-cert-renew.log
```

> [!NOTE]
> Make sure to change "yourdomain.com" to your actual domain. Paths may vary (e.g., `/etc/pki/tls/` on RHEL/CentOS).

Save, exit and make it executable:

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/iredmail-cert-update.sh
```

### 3. Link Hook to Certbot

Edit the domain renewal configuration:

```bash
sudo nano /etc/letsencrypt/renewal/yourdomain.com.conf
```

Add under `[renewalparams]`:

``` bash
deploy_hook = /etc/letsencrypt/renewal-hooks/deploy/iredmail-cert-update.sh
```

### 4. Test the Renewal

Simulate a renewal using Certbot's dry-run parameter:

```bash
sudo certbot renew --dry-run
```

Check that the services restart:

```bash
sudo systemctl status dovecot # replace dovecot with postfix / nginx for the other services
```

Example output (check the date/time noted):

```bash
dovecot.service - Dovecot IMAP/POP3 email server
     Loaded: loaded (/usr/lib/systemd/system/dovecot.service; enabled; preset: enabled)
     Active: active (running) since Thu 2025-03-13 11:32:18 UTC; 1h 10min ago
       Docs: man:dovecot(1)
             https://doc.dovecot.org/
   Main PID: 195567 (dovecot)
     Status: "v2.3.21 (47349e2482) running"
      Tasks: 10 (limit: 4654)
     Memory: 12.3M (peak: 12.8M)
        CPU: 110ms
     CGroup: /system.slice/dovecot.service
         <snip>
```

### 5. Validation

Check that the Certbot timer is active and running:
`sudo systemctl status snap.certbot.renew.timer`
Verify the issued certificate:
`openssl s_client -connect yourdomain.com:993`
Look for service restarts in the logs:
`sudo journalctl -u nginx -u postfix -u dovecot --since "5 minutes ago"`
Check the systemd timers:
`sudo systemctl list-timers | grep certbot`

### 6. Troubleshooting

Common issues with renewals include:

- Slow DNS Propagation - Increase the certbot timer (e.g. --dns-cloudflare-propagation-seconds 30)
- Services not restarting as expected - Check the hook permissions: `ls -l /etc/letsencrypt/renewal-hooks/deploy`
- Timer not running? - Check that it is enabled: `sudo systemctl enable snap.certbot.renew.timer`
- Incorrect certificate being served? Check the symlinks are correct.
- Snap version incorrect or not updating? Try `sudo snap refresh certbot`.
- Hook script not triggering: Test manually with `sudo certbot renew --deploy-hook /path/to/script --dry-run`. Check the renewal config (`/etc/letsencrypt/renewal/yourdomain.com.conf`) for typos in `deploy_hook`.

> [!TIP]
> For persistent issues, enable verbose Certbot logging: `certbot renew --verbose > /var/log/certbot-renew.log 2>&1`.

### 7. Additional Notes

Prefer cron? Disable the certbot timer - `sudo systemctl disable snap.certbot.renew.timer` and add the following to the *root* crontab (`sudo crontab -e`):

```cron
0 3 * * * /snap/bin/certbot renew --deploy-hook /etc/letsencrypt/renewal-hooks/deploy/iredmail-cert-update.sh
```

## Conclusion

You now have a fully automated Let's Encrypt certificate renewal for iRedMail using Certbot's `systemd` timer. For suggestions, comments or issues, please open a GitHub issue at [<link to follow - check back for updates>] or comment below!

## References

- [iRedMail Let’s Encrypt Article](https://docs.iredmail.org/letsencrypt.html)
- [Certbot Renewing Certificates](https://eff-certbot.readthedocs.io/en/stable/using.html#renewing-certificates)
- [Certbot DNS Plugins](https://eff-certbot.readthedocs.io/en/latest/using.html#dns-plugins)
