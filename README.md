# How to Deploy a Website to Digital Ocean 💧 LAMP + rsync + DNS Setup

## How To Use the LAMP 1-Click Install on DigitalOcean

https://www.digitalocean.com/community/tutorials/how-to-use-the-lamp-1-click-install-on-digitalocean

## DNS: Point Your Domain

1. Do this with the domain registrar.
2. Create an A Record to the IP address.
3. Create a CNAME for 'www' with a value or target of 'fiberandkraft.com'.
4. Add the domain name to Digital Ocean.

## Droplet Setup

### SSH in as root

```zsh
# Login as root
ssh root@<IP>
```

### Update + upgrade first

```zsh
apt update && apt upgrade -y

# (Optional) reboot if kernel updates were installed:
reboot

# reconnect:
ssh root@<IP>
```

### Check what’s installed

```zsh
apache2 -v
mysql --version
php -v
systemctl status apache2
```

### Check the Firewall

This is the expected output:

| To               | Action | From          |
| ---------------- | ------ | ------------- |
| 22/tcp           | LIMIT  | Anywhere      |
| Apache Full      | ALLOW  | Anywhere      |
| 22/tcp (v6)      | LIMIT  | Anywhere (v6) |
| Apache Full (v6) | ALLOW  | Anywhere (v6) |

```zsh
ufw status

# if inactive
ufw allow OpenSSH
ufw allow 80
ufw allow 443
ufw enable
```

### Create your non-root sudo user

You can leave the questions blank when creating a sudo user:

-   Full Name []:
-   Room Number []:
-   Work Phone []:
-   Home Phone []:
-   Other []:
-   Is this information correct? [Y/n]

```zsh
adduser angela
usermod -aG sudo angela
```

### Copy root’s authorized key to the new user so you can SSH without passwords

```zsh
# make the .ssh
mkdir -p /home/angela/.ssh

# copy the keys from root
cp -a /root/.ssh/authorized_keys /home/angela/.ssh/

# change the ownership + read/write/execute permissions
chown -R angela:angela /home/angela/.ssh
chmod 700 /home/angela/.ssh
chmod 600 /home/angela/.ssh/authorized_keys
```

### Disable root Login with SSH

To exit the nano editor and save:

```zsh
ctrl + x
Y enter
enter (again)
```

```zsh
nano /etc/ssh/sshd_config
# Set/confirm:
#   PermitRootLogin no
#   PasswordAuthentication no
systemctl reload ssh
```

### Reconnect as the new user

```zsh
ssh angela@fiberandkraft.com
```

### Add sudo user to the www-data group

```zsh
sudo usermod -aG www-data angela

# logout + log back in for the group change to take effect
exit
ssh angela@fiberandkraft.com

# confirm it worked
groups

# you should see
angela sudo www-data
```

### Fix the permissions

That 775 gives group write access, so angela (as part of the www-data group) can update files without breaking Apache ownership.

```zsh
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 775 /var/www/html

# If you want to make sure all future files keep that group:
sudo chmod g+s /var/www/html
# That sets the setgid bit, so any new files/folders inherit the www-data group automatically.
```

### Deploy via rsync from your local machine

#### Just remember that the trailing slash in your local directory path matters:

-   `/Users/angelajholden/Projects/coming-soon/` (with `/`) syncs contents of that folder into /var/www/html/.
-   Without `/`, it would nest the folder inside (`/var/www/html/coming-soon/`).

```zsh
# rsync flags: -a archive, -z compress, -P progress/partial, --delete keeps remote in sync.
rsync -avz --progress --delete --exclude '.git' --exclude '.gitignore' --exclude '.github' --exclude '.DS_Store' --exclude 'README.md' --exclude 'LICENSE.md' /Users/angelajholden/Projects/knit-picks-clone/ angela@fiberandkraft.com:/var/www/html/

# Just in case you need to reset ownership/permissions after rsync:
sudo chown -R www-data:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

## Let's Encrypt

### Make sure DNS is ready

```zsh
ping fiberandkraft.com
# crtl + c to quit
```

### Check that Certbot is installed

```zsh
# The 1-Click LAMP image should already have it
certbot --version

# If you get a version number, you’re set.
# If not (rare), install it manually:
sudo apt install certbot python3-certbot-apache -y
```

### Request and install the certificate

We have to install an SSL certificate on both the root and www, even if we're just doing a redirect to the root.

Browsers are checking the domain on port 443 first, and if the SSL/TLS handshake fails (no certificate) then it can't complete the redirect. Make sure you install the SSL certificate on both versions of the domain. It's free!

```zsh
# Run Certbot’s Apache plugin:
sudo certbot --apache -d fiberandkraft.com -d www.fiberandkraft.com
```

#### Certbot will:

1. Ask for your email (for renewal notices)
2. Ask to agree to the terms
3. Automatically edit Apache to use HTTPS
4. Reload Apache

```zsh
# When it’s done, you’ll see something like:
Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/fiberandkraft.com/fullchain.pem
```

### Enable SSL site if needed

```zsh
sudo a2ensite 000-default-le-ssl.conf
sudo systemctl reload apache2
```

### Run this if in a redirect loop

```zsh
sudo a2enmod ssl
sudo systemctl reload apache2
```

### Auto-renewal check

```zsh
# Certbot installs a renewal timer automatically.
# It checks twice per day, ~12 hours
# If it's within 30 days of renewal, it renews automatically
# Verify it
sudo systemctl list-timers | grep certbot

# Manual dry-run
sudo certbot renew --dry-run

# Output should end with
Congratulations, all renewals succeeded.
```

### Verify which domains are active

```zsh
sudo certbot certificates

# You’ll see a list like:
Certificate Name: fiberandkraft.com
Domains: fiberandkraft.com www.fiberandkraft.com
Expiry Date: 2026-01-20
```

## Edit the Apache Virtual Host Files

### Port 80 VHost

```zsh
sudo nano /etc/apache2/sites-available/000-default.conf
```

Add this INSIDE `<VirtualHost *:80> </VirtualHost>`, at the top of the page.

```zsh
ServerName fiberandkraft.com
ServerAlias www.fiberandkraft.com
Redirect 301 / https://fiberandkraft.com/
```

Comment out these four rewrite lines at the bottom of the file:

```zsh
# RewriteEngine on
# RewriteCond %{SERVER_NAME} =www.fiberandkraft.com [OR]
# RewriteCond %{SERVER_NAME} =fiberandkraft.com
# RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
```

### Port 443 VHost

```zsh
sudo nano /etc/apache2/sites-available/000-default-le-ssl.conf
```

The port 443 vhost file should look like this:

```zsh
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ServerName fiberandkraft.com
    ServerAlias www.fiberandkraft.com

    RewriteEngine On
    RewriteCond %{HTTP_HOST} ^www\.fiberandkraft\.com$ [NC]
    RewriteRule ^ https://fiberandkraft.com%{REQUEST_URI} [R=301,L]

    <Directory /var/www/html/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <IfModule mod_dir.c>
        DirectoryIndex index.php index.pl index.cgi index.html index.xhtml index.htm
    </IfModule>

    Include /etc/letsencrypt/options-ssl-apache.conf
    SSLCertificateFile /etc/letsencrypt/live/fiberandkraft.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/fiberandkraft.com/privkey.pem
</VirtualHost>
</IfModule>
```

### Make sure the rewite module is enabled and reload Apache

```zsh
sudo a2enmod rewrite
sudo systemctl reload apache2

# If you want to be extra sure the SSL vhost is active:
sudo a2ensite 000-default-le-ssl.conf
sudo systemctl reload apache2
```

### Test the redirects

```zsh
# Then you can confirm the redirect behavior later in your browser or by running:
curl -I http://fiberandkraft.com
curl -I http://www.fiberandkraft.com
curl -I https://fiberandkraft.com
curl -I https://www.fiberandkraft.com

# You should see:
HTTP/1.1 301 Moved Permanently
Location: https://fiberandkraft.com/
```

### Test the result

🔒 You should see the lock icon and your site.

```zsh
# Open your site in the browser
http://fiberandkraft.com
http://www.fiberandkraft.com
https://fiberandkraft.com
https://www.fiberandkraft.com
```

## Links

DigitalOcean Tutorials

https://www.digitalocean.com/community/tutorials

How To Use the LAMP 1-Click Install on DigitalOcean

https://www.digitalocean.com/community/tutorials/how-to-use-the-lamp-1-click-install-on-digitalocean

How to Create SSH Keys with OpenSSH on MacOS or Linux

https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/create-with-openssh/

---

### Setup

How To Install LAMP Stack (Apache, MySQL, PHP) on Ubuntu

https://www.digitalocean.com/community/tutorials/how-to-install-lamp-stack-on-ubuntu

How To Set Up an Ubuntu Server on a DigitalOcean Droplet

https://www.digitalocean.com/community/tutorials/how-to-set-up-an-ubuntu-server-on-a-digitalocean-droplet

---

### Sudo Users

How To Create a New Sudo-Enabled User on Ubuntu

https://www.digitalocean.com/community/tutorials/how-to-create-a-new-sudo-enabled-user-on-ubuntu

How To Edit the Sudoers File Safely

https://www.digitalocean.com/community/tutorials/how-to-edit-the-sudoers-file

---

### Rsync

How To Use Rsync to Sync Local and Remote Directories

https://www.digitalocean.com/community/tutorials/how-to-use-rsync-to-sync-local-and-remote-directories

---

### SFTP

How To Use SFTP to Securely Transfer Files with a Remote Server

https://www.digitalocean.com/community/tutorials/how-to-use-sftp-to-securely-transfer-files-with-a-remote-server

How to Transfer Files to Droplets With FileZilla

https://docs.digitalocean.com/products/droplets/how-to/transfer-files/
