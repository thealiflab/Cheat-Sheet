# <img src="assets/Compute/Lightsail.svg" width="48" height="48"/> &nbsp;SSL Setup: Lightsail WordPress Blueprint + Certbot

Step-by-step guide for provisioning an AWS Lightsail WordPress instance and issuing a free Let's Encrypt SSL certificate with Certbot, including the www → apex redirect and `wp-config.php` fixes needed after the certificate is installed.

---

## Prerequisites
- Domain already registered and Route 53 hosted zone exists with A records pointing to the static IP
- Lightsail instance running with static IP attached
- Ports 80 and 443 open in the Lightsail firewall

---

## Step 1: Create Lightsail Instance
1. Lightsail console > **Create instance**
2. **Linux/Unix** > **WordPress** > **Lightsail blueprint** (not Bitnami)
3. Attach a static IP after creation

---

## Step 2: Route 53 DNS Records
In **Route 53 > Hosted zones > yourdomain.com**, confirm these records exist:

| Record name | Type | Value |
|---|---|---|
| (blank/apex) | A | static IP |
| www | A | static IP |

---

## Step 3: Set ServerName in Apache
SSH into the instance and run:

```bash
sudo nano /etc/apache2/sites-enabled/000-default.conf
```

Add these two lines inside `<VirtualHost *:80>` right after the opening tag:

```apache
ServerName yourdomain.com
ServerAlias www.yourdomain.com
```

Save with `Ctrl+X`, `Y`, `Enter` (as we opened this with nano).

Then test and reload:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

---

## Step 4: Verify DNS is Resolving

```bash
curl -s http://yourdomain.com
```

Must return HTML before proceeding. If it hangs or returns nothing, DNS is not ready yet — wait and retry.

---

## Step 5: Install Certbot for SSL Certificate

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-apache
```

---

## Step 6: Run Certbot

```bash
sudo certbot --apache \
  -d yourdomain.com \
  -d www.yourdomain.com
```

- Select both domains when prompted: `1 2`
- If asked which vhost to use: choose `000-default.conf`
- When asked about redirect: choose `2` (Redirect)

If Certbot cannot auto-configure Apache and says to run `certbot install`, run:

```bash
sudo certbot install --cert-name yourdomain.com
```

Choose `2` (Redirect) when prompted.

---

## Step 7: Add www → apex Redirect

```bash
sudo nano /etc/apache2/sites-available/000-default-le-ssl.conf
```

The file will have one `VirtualHost` block with a `ServerAlias`. Change it to two separate blocks:

```apache
<VirtualHost *:443>
    ServerName yourdomain.com
    DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
    Include /etc/letsencrypt/options-ssl-apache.conf
    SSLCertificateFile /etc/letsencrypt/live/yourdomain.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem
</VirtualHost>

<VirtualHost *:443>
    ServerName www.yourdomain.com
    Redirect permanent / https://yourdomain.com
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/yourdomain.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
```

Save, then:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

---

## Step 8: Fix wp-config.php

First find it:

```bash
find /var/www -name "wp-config.php" 2>/dev/null
```

If it is in `/var/www/` instead of `/var/www/html/`, move it:

```bash
sudo mv /var/www/wp-config.php /var/www/html/wp-config.php
```

Then open it:

```bash
sudo nano /var/www/html/wp-config.php
```

Find these dynamic lines:

```php
define( 'WP_HOME', 'http://' . $_SERVER['HTTP_HOST'] . '/' );
define( 'WP_SITEURL', 'http://' . $_SERVER['HTTP_HOST'] . '/' );
```

Replace with:

```php
define( 'WP_HOME', 'https://yourdomain.com' );
define( 'WP_SITEURL', 'https://yourdomain.com' );
```

Save with `Ctrl+X`, `Y`, `Enter`.

---

## Step 9: Verify Everything

Test that all four URLs redirect correctly to `https://yourdomain.com`:

```bash
curl -I http://yourdomain.com
curl -I https://www.yourdomain.com
```

Both should return `301` with `Location: https://yourdomain.com`.

Then check `wp-config.php` is updated:

```bash
grep -n "WP_HOME\|WP_SITEURL" /var/www/html/wp-config.php
```

It should show:

```php
define( 'WP_HOME', 'https://yourdomain.com' );
define( 'WP_SITEURL', 'https://yourdomain.com' );
```

---

## Auto-renewal

Certbot on Debian via apt sets up a systemd timer automatically. Test it with:

```bash
sudo certbot renew --dry-run
```

Should return `All simulated renewals succeeded`. Certificate renews automatically every 60 days before the 90-day expiry.
