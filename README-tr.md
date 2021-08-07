# Flarum Kurulum Rehberi

Kurulum ortamında güvenliği sağlamak amacıyla root olmayan bir kullanıcı ekleyin:

```
adduser username 
usermod -aG sudo username
su - username
```

Sistemi güncelleyin. Ardından Apache ve MariaDB'yi kurun:

```
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install apache2 mariadb-server -y
```

PHP'nin PPA depolarını ekleyin ve bağımlılıkları kurun:

```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install php8.0 libapache2-mod-php8.0 php8.0-common php8.0-mbstring php8.0-xmlrpc php8.0-soap php8.0-mysql php8.0-gd php8.0-xml php8.0-cli php8.0-zip php8.0-curl wget zip unzip curl git -y
```

PHP yapılandırma dosyalarını açın:

``````
sudo nano /etc/php/8.0/apache2/php.ini
``````

Ve değişkenleri aşağıdaki şekildeki gibi yapılandırın Eğer satırın başında ";" varsa silin:

```
file_uploads = On
allow_url_fopen = On
memory_limit = 4096M
upload_max_file_size = 150M
max_execution_time = 450
date.timezone = Europe/Istanbul
```

Ardından servisleri başlatın:

```
sudo systemctl start apache2
sudo systemctl start mysql
sudo systemctl enable apache2
sudo systemctl enable mysql
```

MariaDB'yi aşağıdaki gibi yapılandırın:

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

Güvenlik amacıyla veritabanındaki root kullanıcısını silin ve yeniden oluşturun:

```
sudo mysql -u root -p
	DROP USER 'root'@'localhost';
	CREATE USER 'root'@'%' IDENTIFIED BY 'password';
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
	FLUSH PRIVILEGES;
	EXIT;
```

Sonra Flarum için veritabanı oluşturun:

```
mysql -u root -p
	CREATE DATABASE dbname;
	GRANT ALL PRIVILEGES ON dbname.* TO 'username'@'localhost' IDENTIFIED BY 'password';
	FLUSH PRIVILEGES;
	EXIT;
```

Composer'u yükleyin:

```
cd ~
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

Flarum kurulum dizininin izinlerini 777 olarak ayarlayın:

```
sudo chmod -R 777 /var/www/html/
```

Sonra `cd` komutu ile dizine girin ve her şeyi silin (dikkatle kullanın!) ve Flarum'u kurun:

```
cd /var/www/html
sudo rm -rf *
#composer config -g repo.packagist composer https://packagist.laravel-china.org
#use repo above in China if official repo is too slow
composer create-project flarum/flarum .
```

Sonra izinleri 755 olarak değiştirin:

```
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

Apache yapılandırma dosyasını açın:

```
sudo nano /etc/apache2/sites-available/000-default.conf
```

Ve aşağıdaki yapılandırmayı `<VirtualHost>` içerisine ekleyin:

```
<Directory "/var/www/html">
AllowOverride All
</Directory>
```

Dosyayı kaydedip çıkın.

Sonra `rewrite`'ı etkinleştirin ve Apache'yi yeniden başlatın:

```
sudo a2enmod rewrite
sudo systemctl restart apache2
```

# İsteğe bağlı:

## Ubuntu LTS'in en güncel sürümüne güncelleyin

Önce *update-manager-core*'u kurun:

```
sudo apt-get install update-manager-core
```

*/etc/update-manager/release-upgrades* dosyasının *Prompt* satırında LTS sürüm için *lts*, Normal sürüm için *normal* olarak ayarlandığından emin olun.

```
cat /etc/update-manager/release-upgrades
```

Sonra sistem güncellemesini başlatın.

```
sudo do-release-upgrade
```

## Düşük belleğiniz varsa bir takas alanı oluşturun

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

## Sitenizde HTTPS'i etkinleştirin

Let's Encrypt'i sisteme indirin. Ardından istemciyi güncelleyin ve bağımlıkları kurun.

```
git clone https://github.com/letsencrypt/letsencrypt && cd letsencrypt
sudo ./letsencrypt-auto --help
```

Şimdi 443 numaralı bağlantı noktasının açık olduğundan emin olun, ardından aşağıdakileri çalıştırarak sertifika alabilirsiniz.

```
sudo ./letsencrypt-auto --apache -d your.domain.com
```

Sonra Flarum'un yapılandırma dosyasını açın.

```
sudo nano config.php
#bu komutu Flarum'un kök dizini (kurulum yaptığınız dizin) içerisinde çalıştırmalısınız.
```

Ve aşağıdaki şekilde değiştirin.

```
'url' => 'http://your.domain.com'
 	den
'url' => 'https://your.domain.com'
```

.htaccess'i düzenleyin.

```
sudo nano .htaccess
```

Aşağıdaki yapılandırmayı <IfModule mod_rewrite.c>'nin içerisine yazın.

```
#start https redirect                                                                       
RewriteCond %{HTTPS} off                                                                      
RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}                                             
#end https redirect
```

Sonra Apache'yi yeniden başlatın.

```
sudo systemctl restart apache2
```

### Güncelleme

Sertifikalar üç ay sürer (aslında 90 gün), bu nedenle yenilemek için **yalnızca** aşağıdakileri çalıştırmanız gerekir (gerçekte çalışmayı durdurmadan önce bile).

```
sudo ./letsencrypt-auto --apache -d your.domain.com
sudo systemctl restart apache2
```

# Kaynaklar

[https://discuss.flarum.org/d/1623-obtain-an-ssl-certificate-and-run-your-forum-with-https-for-free](https://discuss.flarum.org/d/1623-obtain-an-ssl-certificate-and-run-your-forum-with-https-for-free)

[https://flarum.org/docs/installation/](https://flarum.org/docs/installation/)

[https://jsthon.com/flarum-installation-guide/](https://jsthon.com/flarum-installation-guide/)

[https://www.howtoing.com/ubuntu-flarum](https://www.howtoing.com/ubuntu-flarum)

[https://laravel-china.org/composer](https://laravel-china.org/composer)

[https://askubuntu.com/questions/766334/cant-login-as-mysql-user-root-from-normal-user-account-in-ubuntu-16-04](https://askubuntu.com/questions/766334/cant-login-as-mysql-user-root-from-normal-user-account-in-ubuntu-16-04)

[https://wiki.ubuntu.com/BionicBeaver/ReleaseNotes](https://wiki.ubuntu.com/BionicBeaver/ReleaseNotes)
