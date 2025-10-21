# Deploy a Website w/Digital Ocean 💧 LAMP + rsync + DNS Setup

## 1) SSH in as root

```zsh
ssh root@<IP>
```

## 2) Update + upgrade first

```zsh
apt update && apt upgrade -y
# (Optional) reboot if kernel updates were installed:
reboot
# reconnect:
ssh root@<IP>
```

# 3) Quick firewall baseline

```zsh
ufw allow OpenSSH
ufw enable
ufw status
```

# 4) Create your non-root sudo user

```zsh
adduser angela
usermod -aG sudo angela
```

# 5) Copy root’s authorized key to the new user so you can SSH without passwords:

```zsh
mkdir -p /home/angela/.ssh
cp -a /root/.ssh/authorized_keys /home/angela/.ssh/
chown -R angela:angela /home/angela/.ssh
chmod 700 /home/angela/.ssh
chmod 600 /home/angela/.ssh/authorized_keys
```

# 6) Now harden SSH:

```zsh
nano /etc/ssh/sshd_config
# Set/confirm:
#   PermitRootLogin no
#   PasswordAuthentication no
systemctl restart ssh
```

# 7) Reconnect as the new user:

```zsh
ssh angela@<IP>
```

# 8) Install LAMP (Apache, MySQL, PHP)

```zsh
sudo apt install -y apache2
sudo ufw allow 'Apache'
# or, if you plan for HTTPS too:
# sudo ufw allow 'Apache Full'
sudo systemctl status apache2

sudo apt install -y mysql-server
sudo mysql_secure_installation

sudo apt install -y php libapache2-mod-php php-mysql
sudo systemctl restart apache2
# Test in browser at http://<IP> — you should see Apache default page.
```

# 9) Drop in a “coming soon” page

```zsh

```

# 10) Deploy via rsync from your local machine

```zsh
# Rsync flags: -a archive, -z compress, -P progress/partial, --delete keeps remote in sync.

# Then on the server, set sane ownership/permissions:
sudo chown -R www-data:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

# 11) Open HTTP/HTTPS + verify

```zsh
sudo ufw allow 80
sudo ufw allow 443
sudo ufw status
```

# 12) DNS: point your domain

```zsh
ssh angela@demo.yourdomain.com
```

## Links

DigitalOcean Tutorials

https://www.digitalocean.com/community/tutorials

### Setup

How To Install LAMP Stack (Apache, MySQL, PHP) on Ubuntu

https://www.digitalocean.com/community/tutorials/how-to-install-lamp-stack-on-ubuntu

How To Set Up an Ubuntu Server on a DigitalOcean Droplet

https://www.digitalocean.com/community/tutorials/how-to-set-up-an-ubuntu-server-on-a-digitalocean-droplet

### Sudo Users

How To Create a New Sudo-Enabled User on Ubuntu

https://www.digitalocean.com/community/tutorials/how-to-create-a-new-sudo-enabled-user-on-ubuntu

How To Edit the Sudoers File Safely

https://www.digitalocean.com/community/tutorials/how-to-edit-the-sudoers-file

### Rsync

How To Use Rsync to Sync Local and Remote Directories

https://www.digitalocean.com/community/tutorials/how-to-use-rsync-to-sync-local-and-remote-directories

### SFTP

How To Use SFTP to Securely Transfer Files with a Remote Server

https://www.digitalocean.com/community/tutorials/how-to-use-sftp-to-securely-transfer-files-with-a-remote-server

How to Transfer Files to Droplets With FileZilla

https://docs.digitalocean.com/products/droplets/how-to/transfer-files/

### Misc

How To Use the LAMP 1-Click Install on DigitalOcean

https://www.digitalocean.com/community/tutorials/how-to-use-the-lamp-1-click-install-on-digitalocean

How to Create SSH Keys with OpenSSH on MacOS or Linux

https://docs.digitalocean.com/products/droplets/how-to/add-ssh-keys/create-with-openssh/
