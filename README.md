# WSL For Web-server
Stack:
<p>
  <img src="https://img.shields.io/badge/Nginx-blue" alt="Nginx">
  <img src="https://img.shields.io/badge/MySQL-blue" alt="MySQL">
  <img src="https://img.shields.io/badge/PHP-FPM-blue" alt="PHP-FPM">
</p>
This guide is primarily aimed at users who have a basic understanding of Linux and are familiar with the technology stack in use.

## Installing the necessary packages
Assuming you have just configured WSL on your system and the version of Ubuntu you want.

### Nginx

Install and run Nginx
```bash
$ sudo apt update
$ sudo apt install nginx
$ sudo /etc/init.d/nginx start
```

If you have done everything correctly, the link http://127.0.0.1/ should display the “Welcome to nginx!” start page in your browser.

Let's stop here with nginx for now, we'll come back at the site configuration stage.

### MySQL

```bash
$ sudo apt install mysql-server
$ sudo /etc/init.d/mysql start
```
Even though we have a local site, we need to get used to the right thing right away - we run the standard script for configuring MySQL security policies (setting passwords, validation requirements, anonymous access and all that):

```bash
$ sudo mysql_secure_installation
```

### PHP-FPM

```bash
$ sudo apt install php-fpm php-mysql
$ sudo /etc/init.d/php7.2-fpm start
```
## Host Configuration
### Nginx configuration
Go to the folder with nginx site configs and create your own on the basis of the default config. It is best to name the config files by domain, and for local projects use a separate domain zone, so that there is no confusion whether the project is opened locally or on a remote server. I usually use the zone `.local` .

```bash
$ cd /etc/nginx/sites-available/
$ sudo cp default blog.local
```
Edit the config:
```bash
$ sudo nano blog.local
```
```nginx
server {
    listen 80;
    root /mnt/f/my/web/blog/public;
    index index.php index.html index.htm index.nginx-debian.html;
    server_name blog.local;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        fastcgi_buffering off;
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
