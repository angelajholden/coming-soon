# Deploy a Website w/Digital Ocean 💧

## LAMP + rsync + DNS Setup

### 1. SSH in as root

```zsh
ssh root@<IP>
```

### 2. Update + upgrade first

```zsh
apt update && apt upgrade -y

# (Optional) reboot if kernel updates were installed:
reboot

# reconnect:
ssh root@<IP>
```

### 3. Check What’s Installed

```zsh
apache2 -v
mysql --version
php -v
ufw status
systemctl status apache2
```

### 4. Create your non-root sudo user

```zsh
adduser angela
usermod -aG sudo angela
```

### 5. Copy root’s authorized key to the new user so you can SSH without passwords

```zsh
mkdir -p /home/angela/.ssh
cp -a /root/.ssh/authorized_keys /home/angela/.ssh/
chown -R angela:angela /home/angela/.ssh
chmod 700 /home/angela/.ssh
chmod 600 /home/angela/.ssh/authorized_keys
```

### 6. Now harden SSH

```zsh
nano /etc/ssh/sshd_config
# Set/confirm:
#   PermitRootLogin no
#   PasswordAuthentication no
systemctl restart ssh
```

### 7. Reconnect as the new user

```zsh
ssh angela@<IP>
```

### 8. Deploy via rsync from your local machine

#### Just remember that the trailing slash in your local directory path matters:

-   `/Users/angelajholden/Projects/coming-soon/` (with `/`) syncs contents of that folder into /var/www/html/.
-   Without `/`, it would nest the folder inside (`/var/www/html/coming-soon/`).

```zsh
# Rsync flags: -a archive, -z compress, -P progress/partial, --delete keeps remote in sync.
rsync -avz --progress --delete \
  --exclude '.git' \
  --exclude '.gitignore' \
  --exclude '.github' \
  --exclude '.DS_Store' \
  --exclude 'README.md' \
  --exclude 'LICENSE.md' \
  /Users/angelajholden/Projects/coming-soon/ \
  root@<IP>:/var/www/html/

# Then on the server, set sane ownership/permissions:
sudo chown -R www-data:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

### 9. DNS: point your domain

1. Do this with the domain registrar.
2. Point the domain A Record to the IP address.
3. Add the domain name to Digital Ocean.

```zsh
ssh angela@fiberandkraft.com
```

## Let's Encrypt

### 1. Make sure DNS is ready

```zsh
ping fiberandkraft.com
```

### 2. Check that Certbot is installed

```zsh
# The 1-Click LAMP image should already have it
certbot --version

# If you get a version number, you’re set.
# If not (rare), install it manually:
sudo apt install certbot python3-certbot-apache -y
```

### 3. Request and install the certificate

```zsh
# Run Certbot’s Apache plugin:
sudo certbot --apache -d fiberandkraft.com -d www.fiberandkraft.com
```

#### Certbot will:

1. Ask for your email (for renewal notices)
2. Ask to agree to the terms
3. Detect your domain(s) from Apache config — you can select your “coming soon” site
4. Automatically edit Apache to use HTTPS
5. Reload Apache

```zsh
# When it’s done, you’ll see something like:
Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/demo.yourdomain.com/fullchain.pem
```

#### Certbot will also:

-   Request certificates for both the root domain and 'www'
-   Add the redirect automatically so www → root over HTTPS

```zsh
# Then you can confirm the redirect behavior later in your browser or by running:
curl -I http://www.fiberandkraft.com

# You should see:
HTTP/1.1 301 Moved Permanently
Location: https://fiberandkraft.com/
```

### 4. Test the result

```zsh
# Open your site in the browser
https://demo.yourdomain.com
```

🔒 You should see the lock icon and your site.

```zsh
# Or test in terminal
curl -I https://fiberandkraft.com
# Look for HTTP/2 200 — that means it’s serving over HTTPS.
```

### 5. Auto-renewal check

```zsh
# Certbot installs a renewal timer automatically.
# Verify it
sudo systemctl list-timers | grep certbot

# Manual dry-run
sudo certbot renew --dry-run

# Output should end with
Congratulations, all renewals succeeded.
```

### 6. Optional hardening

-   Force HTTPS redirect (Certbot usually adds this)
-   Check firewall allows port 443

```zsh
sudo ufw allow 443
sudo ufw status
```

### 7. Verify which domains are active

```zsh
sudo certbot certificates

# You’ll see a list like:
Certificate Name: fiberandkraft.com
Domains: fiberandkraft.com www.fiberandkraft.com
Expiry Date: 2026-01-20
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
