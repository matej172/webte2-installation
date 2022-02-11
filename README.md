# Webte2 server setup

Nastavenie LEMP webového servera pre predmet webte2 FEI STU

## Softvér a verzie
- Ubuntu 20.04
- Nginx
- PHP 8.1
- MySQL 8
- PhpMyAdmin

## Pripojenie k serveru pomocou SSH

Cez program putty (windows) alebo priamo cez terminál (OSx/Linux) sa pripojiť k svojmu pridelenému serveru.
```sh
ssh username@147.175.YY.XX
```
Po prompte zadať heslo.

## Update systému
Po prihlásení je nutné systém aktualizovať
```sh
sudo apt update
sudo apt dist-upgrade
```
Pridanie repozitárov pre novšie verzie softvéru php a phpmyadmin
```sh
sudo add-apt-repository ppa:ondrej/php
sudo add-apt-repository ppa:phpmyadmin/ppa
sudo apt update
```
## Nginx
Inštalácia balíkov webserveru nginx, textového editora vim.
```sh
sudo apt install nginx vim
```
Po navštívení IP adresy by webový prehliadač mal zobrazovať
![nginx](https://raw.githubusercontent.com/matej172/webte2-installation/main/img/nginx.png)

Pridanie usera do skupiny www-data.
```sh
sudo usermod -aG www-data username
```
Zmena sa prejaví až pri novej relácii, systém je možné rovno reštartovať.
```sh
sudo reboot
```
Reštart systému ukončí reláciu, po niekoľkých sekundách treba znova nadviazať [ssh spojenie](#pripojenie-k-serveru-pomocou-ssh).

### Kontrola pridania používateľa do skupiny (Nepovinné)

Po zadaní príkazu

```sh
groups
```
by výstup mal vyzerať
```sh
username sudo www-data
```

## MySQL
Inštalácia MySQL databázového serveru.
```sh
sudo apt install mysql-server
sudo mysql_secure_installation
```
Odpovede na otázky počas konfigurácie:
- no
- heslo na ssh prihlásenie (pravdepodobne je to jedno aké heslo sa zadá)
- remove anonymous user - yes
- Disallow root login remotely? - yes
- Reload privilege tables now? - yes

Pirpojenie ku MySQL konzole.
```sh
sudo mysql
```
Prompt sa zmení na ```mysql>```
Vytvorenie nového používateľa pre prístup a srpávu databáz.
```sh
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';
```
Pridanie privilégií pre prácu s databázami.
```sh
GRANT ALL PRIVILEGES ON *.* TO 'username'@'localhost';
FLUSH PRIVILEGES;
```
Opustenie konzoly MySQL pomocou ```Ctrl + d``` alebo ```exit```.


### Kontrola pridania používateľa pre prístup k databáze (Nepovinné)

Prihlásenie sa do MySQL konzoly pod novým používateľom
```sh
mysql -u username -p
```

## PHP
Inštalácia PHP 8.1.

```sh
sudo apt install php-fpm
```

Odpoveďou na príkaz ```php -v``` by malo byť
```sh
PHP 8.1.2 (cli) (built: Jan 24 2022 10:42:33) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.1.2, Copyright (c) Zend Technologies
    with Zend OPcache v8.1.2, Copyright (c), by Zend Technologies
```

### Vytvorenie Virtual host konfigurácie pre URL
Reťazec **XX** nahradiť prideleným číslom podľa URL

```sh
sudo vim /etc/nginx/sites-available/siteXX.webte.fei.stuba.sk
```
Do súboru vložiť obsah a zameniť režazec **XX** za posledný číselný segment priradenej IP adresy:

```sh
server {
       listen 80;
       listen [::]:80;

       server_name siteXX.webte.fei.stuba.sk;

       root /var/www/siteXX.webte.fei.stuba.sk;
       index index.html index.php;
       
       location / {
               try_files $uri $uri/ =404;
       }
       
       location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
       }
}
```
> Poznámka: Príkaz `:%s/XX/16/g` vo Vim nahradí všetky výskyty reťazca XX číslom 16.

Vytvorenie symbolického odkazu súboru.
```sh
sudo ln -s /etc/nginx/sites-available/siteXX.webte.fei.stuba.sk /etc/nginx/sites-enabled/
```

Po spustení príkazu ```sudo service nginx restart``` by mal webový prehliadač ukazovať  chybu 404.
### Vytvorenie adresára pre webový server
Skripty a súbory v tomto adresáre sa zobrazia po navštívení pridelenej domény.

Aby bolo možné vytvárať zápisy do adresára musí patriť skupine www-data a mať prístup na zápis pre skupinu.
```sh
cd /var
sudo chown -R www-data:www-data www/
sudo chmod g+w -R www/
```
Po zmene oprávnení je možné zapisovať do adresára ```/var/www``` bez sudo privilégií.
```sh
cd /var/www
mkdir siteXX.webte.fei.stuba.sk
cd siteXX.webte.fei.stuba.sk
vim index.php
```

Vytvorte jednoduchý PHP skript, napr.
```php
<?php
	echo "Hello world!"
?>
```
Po navštívení pridellenej URL by sa mala načítať prázdna stránka s textom _Hello world!_.

### SSL certifikát pre HTTPS

Stiahnuť súbory:
- webte.fei.stuba.sk-chain-cert.pem
- webte.fei.stuba.sk.key

Skopírovať ich do:
- /etc/ssl/certs/webte.fei.stuba.sk-chain-cert.pem;
- /etc/ssl/private/webte.fei.stuba.sk.key;

Zmeniť konfiguráciu Nginx v súbore ```/etc/sites-available/siteXX.webte.fei.stuba.sk```

```sh
server {
       listen 80;
       listen [::]:80;

       server_name siteXX.webte.fei.stuba.sk;

       rewrite ^ https://$server_name$request_uri? permanent;
}

server {
        listen 443 ssl;
        listen [::]:443 ssl;

        server_name siteXX.webte.fei.stuba.sk;

        access_log /var/log/nginx/access.log;
        error_log  /var/log/nginx/error.log info;

        root /var/www/siteXX.webte.fei.stuba.sk;
        index index.php index.html;

        ssl on;
        ssl_certificate /etc/ssl/certs/webte.fei.stuba.sk-chain-cert.pem;
        ssl_certificate_key /etc/ssl/private/webte.fei.stuba.sk.key;

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        }
}
```

Reštartovať Nginx príkazom
```sh
sudo service nginx restart
```

Po navštívení pridelenej domény vo webovom prehliadači sa stránka načíta pomcou https protokolu
![nginx](https://raw.githubusercontent.com/matej172/webte2-installation/main/img/ssl.png)

## PhpMyAdmin

Inštalácia GUI utility pre správu databázy cez prehliadač.

```sh
sudo apt install phpmyadmin
```

Vytvoriť súbor ```/etc/nginx/snippets/phpmyadmin.conf``` a vložiť obsah:

```sh
location /phpmyadmin {
    root /usr/share/;
    index index.php index.html index.htm;
    location ~ ^/phpmyadmin/(.+\.php)$ {
        try_files $uri =404;
        root /usr/share/;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include /etc/nginx/fastcgi_params;
    }

    location ~* ^/phpmyadmin/(.+\.(jpg|jpeg|gif|css|png|js|ico|html|xml|txt))$ {
        root /usr/share/;
    }
}
```

Do konfiguračného súboru ```/etc/nginx/sites-available/siteXX.webte.fei.stuba.sk ``` pridať riadok

```sh
include snippets/phpmyadmin.conf;
```

### Inštalácia verzie 5.1.2

V prípade, že aplikácia phpMyAdmin po prihlásení vracia chybu 500, je nutné urobiť aktualizáciu na novšiu verziu.

```sh
cd ~
wget www.phpmyadmin.net/downloads/phpMyAdmin-latest-all-languages.zip
sudo apt install unzip
unzip phpMyAdmin-latest-all-languages.zip
sudo rm -R /usr/share/phpmyadmin/
sudo mkdir /usr/share/phpmyadmin
sudo mv phpMyAdmin-*/* /usr/share/phpmyadmin/
```

#### phpMyAdmin blowfish secret generator
```sh
sudo cp /usr/share/phpmyadmin/config.sample.inc.php /usr/share/phpmyadmin/config.inc.php
sudo vim /usr/share/phpmyadmin/config.inc.php
```
Otvoriť stránku [https://phpsolved.com/phpmyadmin-blowfish-secret-generator/](https://phpsolved.com/phpmyadmin-blowfish-secret-generator/ )

Prepísať riadok ```$cfg['blowfish_secret'] = ''``` podľa zobrazenej stránky.


#### Vytvorenie tmp adresára pre phpMyAdmin
```sh
cd /usr/share/phpmyadmin/
sudo mkdir tmp
sudo chmod 777 tmp/
```


## License

MIT
