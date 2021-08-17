# Flarum Setup Guide

Add non-root user to ensure safety in production environment:

```
adduser username 
usermod -aG sudo username
su - username
```

Update+upgrade and install Apache+MariaDB:

```
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install apache2 mariadb-server -y
```

Add PHP PPA and install dependencies:

```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install php8.0 libapache2-mod-php8.0 php8.0-common php8.0-mbstring php8.0-xmlrpc php8.0-soap php8.0-mysql php8.0-gd php8.0-xml php8.0-cli php8.0-zip php8.0-curl wget zip unzip curl git -y
```

Edit PHP configuration:

``````
sudo nano /etc/php/8.0/apache2/php.ini
``````

And make sure the following are set to the correct value:

```
file_uploads = On
allow_url_fopen = On
memory_limit = 4096M
upload_max_file_size = 150M
max_execution_time = 450
date.timezone = Asia/Kolkata
```

Start Apache and MySQL services:

```
sudo systemctl start apache2
sudo systemctl start mariadb
sudo systemctl enable apache2
sudo systemctl enable mariadb
```

Then add security measures to MariaDB:

```
sudo mysql_secure_installation
	Enter current password for root (enter for none): Enter
	Set root password? [Y/n]: Y
	New password: 
	Re-enter new password: 
	Remove anonymous users? [Y/n]: Y
	Disallow root login remotely? [Y/n]: Y
	Remove test database and access to it? [Y/n]: Y
	Reload privilege tables now? [Y/n]: Y
```

Recreate root user for MariaDB so non-root users can access it too:

```
sudo mysql -u root -p
	DROP USER 'root'@'localhost';
	CREATE USER 'root'@'%' IDENTIFIED BY 'password';
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
	FLUSH PRIVILEGES;
	EXIT;
```

Then create database and corresponding user for Flarum:

```
mysql -u root -p
	CREATE DATABASE dbname;
	GRANT ALL PRIVILEGES ON dbname.* TO 'username'@'localhost' IDENTIFIED BY 'password';
	FLUSH PRIVILEGES;
	EXIT;
```

After that, install composer:

```
cd ~
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

**If it's too slow in China, use the following instead:**

```
wget https://dl.laravel-china.org/composer.phar -O /usr/local/bin/composer
chmod a+x /usr/local/bin/composer
```

Give 777 perm to non-root user for Flarum install directory:

```
sudo chmod -R 777 /var/www/html/
```

cd the install directory and remove all existing files(use caution!) and install Flarum:

```
cd /var/www/html
sudo rm -rf *
#composer config -g repo.packagist composer https://packagist.laravel-china.org
#use repo above in China if official repo is too slow
composer create-project flarum/flarum .
```

If Flarum says that directories cannot be written during installation (/var/www/html/flarum is not writable error), run the following command.

```
sudo chmod -r 777 /var/www/html/*
```

Change owner to www-data and change perm to 755 (avoid 777 in production environment) after installing.

```
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

Edit Apache configuration:

```
sudo nano /etc/apache2/sites-available/000-default.conf
```

Add the following snippet in VirtualHost:

```
<Directory "/var/www/html">
AllowOverride All
</Directory>
```

Then save file and exit.

After that, enable rewrite for Apache and restart:

```
sudo a2enmod rewrite
sudo systemctl restart apache2
```

# OPTIONALS:

## Update to latest Ubuntu LTS

First make sure *update-manager-core* is installed

```
sudo apt-get install update-manager-core
```

Then make sure *Prompt* line in */etc/update-manager/release-upgrades* is set to *lts* for LTS version or *normal* for non-LTS version

```
cat /etc/update-manager/release-upgrades
```

And finally start system upgrade

```
sudo do-release-upgrade
```

## Create *swap* if you have small memory

```
free -m
mkdir -p /var/_swap_
cd /var/_swap_
dd if=/dev/zero of=swapfile bs=1M count=6000
#6000 means to create 6gig swap, change this number to whatever you want
mkswap swapfile
swapon swapfile
chmod 600 swapfile
echo "/var/_swap_/swapfile none swap sw 0 0" >> /etc/fstab
free -m
```

## Enable HTTPS for your site

Clone Let's Encrypt repo and install and update the client and its dependencies

```
git clone https://github.com/letsencrypt/letsencrypt && cd letsencrypt
sudo ./letsencrypt-auto --help
```

Now ensure that port 443 is open then we can obtain our certificate by running the following

```
sudo ./letsencrypt-auto --apache -d your.domain.com
```

Then, we are gonna configure Flarum as the following

```
sudo nano config.php
#this should be done in Flarum root directory
```

And change the following line

```
'url' => 'http://your.domain.com'
 	to
'url' => 'https://your.domain.com'
```

Edit .htaccess

```
sudo nano .htaccess
```

Add the following in <IfModule mod_rewrite.c> block

```
#start https redirect                                                                       
RewriteCond %{HTTPS} off                                                                      
RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}                                             
#end https redirect
```

Lastly restart Apache service

```
sudo systemctl restart apache2
```

### Updating

The certificates last for three months (90 days in fact) so you will need to run **only** the following to renew it (even before it actually stops working).

```
sudo ./letsencrypt-auto --apache -d your.domain.com
sudo systemctl restart apache2
```

# References

[https://discuss.flarum.org/d/1623-obtain-an-ssl-certificate-and-run-your-forum-with-https-for-free](https://discuss.flarum.org/d/1623-obtain-an-ssl-certificate-and-run-your-forum-with-https-for-free)

[https://flarum.org/docs/installation/](https://flarum.org/docs/installation/)

[https://jsthon.com/flarum-installation-guide/](https://jsthon.com/flarum-installation-guide/)

[https://www.howtoing.com/ubuntu-flarum](https://www.howtoing.com/ubuntu-flarum)

[https://laravel-china.org/composer](https://laravel-china.org/composer)

[https://askubuntu.com/questions/766334/cant-login-as-mysql-user-root-from-normal-user-account-in-ubuntu-16-04](https://askubuntu.com/questions/766334/cant-login-as-mysql-user-root-from-normal-user-account-in-ubuntu-16-04)

[https://wiki.ubuntu.com/BionicBeaver/ReleaseNotes](https://wiki.ubuntu.com/BionicBeaver/ReleaseNotes)
