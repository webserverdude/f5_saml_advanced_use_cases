# SAML Advanced Use Cases - Part 2: Setting up SimpleSAMLphp

## Preface
SimpleSAMLphp is an application providing support for SAML 2.0. It can be used as SAML Service Provider (SP) and/or Identity Provider (IdP).
For demonstrating my SAML use cases I will use SimpleSAMLphp as SAML Service Provider (SP). In a real world use case the SAML Service Provider (SP) would be an ERP (Enterprise Resource Planning) system such as SAP or a content management systems (CMS) like Microsoft SharePoint.

## Versions used

* Debian 12 
* NGINX 1.22.1
* PHP 8.2.7
* SimpleSAMPphp 2.0.6

## Install NGINX and PHP on Debian
For our use-cases we will only need NGINX and PHP, MariaDB oder MySQL is not required (though supported by SimpleSAMLphp for other scenarios).

### Install & configure required software components
```bash
root@simpleSAMLphp:~# apt install nginx php-fpm php-curl php-xml
```

For the purpose of this lab we will use a self-signed SSL certificate for NGINX.
You can issue a certificate for your web server with the following command.
```bash
root@simpleSAMLphp:~# openssl req -newkey rsa:2048 -nodes -keyout /etc/ssl/private/nginx.key -x509 -days 365 -out /etc/ssl/certs/nginx.pem
```

## Install & configure SimpleSAMLphp
After setting up NGINX and PHP we have to install and configure SimpleSAMLphp.

### Download and install SimpleSAMLphp

Download SimpleSAMLphp from https://simplesamlphp.org/download

Go to the directory where you want to install SimpleSAMLphp and extract the archive file you just downloaded:
(I used `/var/www/simplesamlphp`)

```bash
root@simpleSAMLphp:~# cd ~
root@simpleSAMLphp:~# wget https://github.com/simplesamlphp/simplesamlphp/releases/download/v2.0.6/simplesamlphp-2.0.6.tar.gz
root@simpleSAMLphp:~# tar xzf simplesamlphp-2.0.6.tar.gz 
root@simpleSAMLphp:~# mv simplesamlphp-2.0.6 /var/www/simplesamlphp
```

### Configure NGINX to host SimpleSAMLphp

Next configure NGINX to host your site SimpleSAMLphp. Create a `simplesamlphp` config file in `/etc/nginx/sites-available/`.

```
server {
        listen 443 ssl default_server;
        ssl_certificate        /etc/nginx/certificate.pem;
        ssl_certificate_key    /etc/nginx/key.pem;
        ssl_protocols          TLSv1.3 TLSv1.2;
        ssl_ciphers            EECDH+AESGCM:EDH+AESGCM;

        server_name sp.zerortrust.works;
        
        location ^~ /simplesaml {
                alias /var/www/simplesamlphp/public;

                location ~^(?<prefix>/simplesaml)(?<phpfile>.+?\.php)(?<pathinfo>/.*)?$ {

                        include snippets/fastcgi-php.conf;
                        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
                        fastcgi_param SCRIPT_FILENAME $document_root$phpfile;
                        fastcgi_param SCRIPT_NAME /simplesaml$phpfile;
                        fastcgi_param PATH_INFO $pathinfo if_not_empty;
                }
        }
}
```

Now use the `ln` command to create symlink of this conf file in the `sites-enabled` folder and restart nginx.
Verify the config before you reload the config.

```bash
root@simpleSAMLphp:~# ln -s /etc/nginx/sites-available/simplesamlphp /etc/nginx/sites-enabled/simplesamlphp
root@simpleSAMLphp:/etc/nginx/sites-available# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successfulroot@simpleSAMLphp:/var/www/simplesamlphp/config# nginx -s reload
2023/10/23 21:36:51 [notice] 9644#9644: signal process started
```

### Configure SimpleSAMLphp
All config files are stored in the `config/` folder of your SimpleSAMLphp installation.
Just create a copy from the default files and modify this copy. We start with the `config.php` file.

```bash
root@simpleSAMLphp:/var/www/simplesamlphp/config# cp config.php.dist config.php
root@simpleSAMLphp:/var/www/simplesamlphp/config# nano config.php
```

Modify the following settings in the `config.php` file.

```bash
'baseurlpath' => 'https://your.canonical.host.name/simplesaml/',
'auth.adminpassword' => 'setnewpasswordhere',
'secretsalt' => 'randombytesinsertedhere',
'timezone' => 'Europe/Berlin'
```

Now you need to copy also the `authsources.php.dist` template to `authsources.php`

```bash
root@simpleSAMLphp:/var/www/simplesamlphp/config# cp authsources.php.dist authsources.php
``` 

And at this point you should be able to login to your new SimpleSAMLphp installation.

![simpleSAMLphp_basic_install](assets\simpleSAMLphp_basic_install.png)

### Configure SimpleSAMLphp as a SAML Service Provider

The final step to deploy SimpleSAMLphp as a SAML Service Provider is to modify the `entityID` line in the `authsources.php` file.

```bash
'entityID' => 'https://sp.zerotrust.works/',
```

![simpleSAMLphp_SP_ready](assets\simpleSAMLphp_SP_ready.png)

## Next steps

Next we will setup two use cases using our SimpleSAMLphp server as SAML Service Provider (SP).