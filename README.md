# WSL for Web-server
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
Since we still have WSL 1, we will use the file system of the main system. Logical disks inside WSL are mounted in `/mnt` - you can find out the list of partitions with the command `ls -la`.

If you want to see your site on the same http://127.0.0.1/ or http://localhost/ - leave `server_name` and `listen` values by default. In this case, you will not be able to start several different sites, but you will not have to configure `hosts`.

To make the site work, its config needs to be placed in the `/etc/nginx/sites-enabled/` folder, make [symbolic link](https://en.wikipedia.org/wiki/Symbolic_link#:~:text=In%20computing%2C%20a%20symbolic%20link,by%20specifying%20a%20path%20thereto.):

```bash
$ sudo ln -s /etc/nginx/sites-available/blog.local /etc/nginx/sites-enabled/
```

We also turn off the default site, we don't need it anymore:

```bash
$ sudo unlink /etc/nginx/sites-enabled/default
```
Test the config (should be ok) and update nginx:
```bash
$ sudo nginx -t
$ sudo /etc/init.d/nginx reload
```

### Hosts
In order for the browser to understand our blog.local domain, it must get the ip address of the server associated with this domain from somewhere. On the big Internet, DNS-servers are responsible for this, while we just need to specify the correspondence in `hosts (C:Windows\System32\drivers\etc\hosts)`. 

Add the following line (requires administrator rights):
```
127.0.0.1		blog.local
```

### PHP
If a folder for the project doesn't already exist by this point - it's worth pursuing.

Create `index.php`:

```bash
$ echo '<?php echo "Hello world!";' > /mnt/f/my/web/blog/public/index.php
```
Now open our site in a browser - http://blog.local/. If successful, we will see the text `Hello world!`.

We also have the ability to view all server settings, [phpinfo](https://www.php.net/manual/ru/function.phpinfo.php) gives us that ability:
```bash
$ echo '<?php phpinfo();' > /mnt/f/my/web/blog/public/phpinfo.php
```
Open http://blog.local/phpinfo.php and see your settings. 

**Important!** Do not use this feature on combat servers, and especially do not leave such a file lying around with access for everyone. The output contains versions of all used distributions, this is more than enough information, if not for server hacking, then for a targeted attack using vulnerabilities of specific versions.

### MySQL
The easiest way, especially for some self-written projects, is to put an admin. For those who find it unnecessary, you can create a user for the project and the database directly in the mysql console.

Install phpmyadmin together with the necessary php extensions and restart php-fpm to enable the extensions.
```bash
$ sudo apt install phpmyadmin php-mbstring php-gettext
$ sudo /etc/init.d/php7.2-fpm restart
```
When asked “web-server to reconfigure automatically”, select nothing, press Tab and Ok.
In the “Configuring phpmyadmin” window, select Yes and create a user for the utility.

Now we need to configure nginx, we create another config for the phpmyadmin host:
```bash
$ cd /etc/nginx/sites-available/
$ sudo cp default phpmyadmin
$ sudo nano phpmyadmin
```
Content (admin will be available at http://127.0.0.1/pma):

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html/;

    index index.php index.html index.htm index.nginx-debian.html;
    server_name _;

    location /pma {
        alias /usr/share/phpmyadmin/;
        location ~ \.php$ {
            fastcgi_buffering off;
            fastcgi_pass unix:/run/php/php7.2-fpm.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $request_filename;
            include fastcgi_params;
            fastcgi_ignore_client_abort off;
        }
        location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
            access_log    off;
            log_not_found    off;
            expires 1M;
        }
    }
}
```
Power on the site, check the config and reload nginx:
```bash
$ sudo ln -s /etc/nginx/sites-available/phpmyadmin /etc/nginx/sites-enabled/
$ sudo nginx -t
$ sudo /etc/init.d/nginx reload
```
Open http://127.0.0.1/pma, user `phpmyadmin`, password is the same as you entered during installation.

If suddenly the phpmysql user has limited rights (cannot create the database or other users) - these rights can always be granted manually.

Go to the mysql console:
```bash
$ mysql -u root -p
```

And we issue the right rights (in this case, all possible rights):
```sql
GRANT ALL PRIVILEGES ON *.* TO 'phpmyadmin'@'localhost' WITH GRANT OPTION;
```

Go back to phpmyadmin and log back in, the permissions should be in place. To demonstrate the correct work of all packages, let's add the user `blog` and the database `blog` with the table `test` of arbitrary format (the encoding should be chosen `utf8_general_ci`).

## Service autoloading
To make our services start automatically when ubuntu starts - run the following commands:

```bash
$ sudo update-rc.d nginx defaults
$ sudo update-rc.d php7.2-fpm defaults
$ sudo update-rc.d mysql defaults
```
## Verify correct operation
Connect to the database from php and display the contents of the table test (I don't think it's worth mentioning once again that you can't specify open accesses in the code, smart people invented .env files for this purpose):
```php+HTML
<?php
try {
    $dbh = new PDO('mysql:host=localhost;dbname=blog', 'blog', 'blog');
    foreach($dbh->query('SELECT * from test') as $row) {
        echo "<pre>";
        print_r($row);
        echo "</pre>";
    }
    $dbh = null;
} catch (PDOException $e) {
    print "Error!: " . $e->getMessage() . "<br/>";
    die();
}

```
We open our http://blog.local again and see the contents of the table on the screen. 

As a result, everything works, and we have a familiar linux command line in windows (!) with familiar packages that are used in production. The things for which we used to install extremely slow virtualizers are now available out of the box and can be started even without tambourines. We live in an amazing time!

## If something went wrong
### ERR_NAME_NOT_RESOLVED (unable to access site)
The browser could not find the specified domain. Possible causes:

- you have incorrectly configured the site config;
- you did not specify the domain in hosts or specified it incorrectly;
- you did not make `nginx reload` after creating/modifying the site config - this action must be performed after any changes in the config.

### 502 Bad Gateaway

The error is due to nginx not receiving a response from the backend, in this case php-fpm. 

Possible causes:

- php-fpm is not running;
- incorrect php parameters in nginx host settings;
- an error directly in the php-script.

Your best helpers in troubleshooting any problem are logs:
The error is due to nginx not receiving a response from the backend, in this case php-fpm. 

Possible causes:

- php-fpm is not running;
- incorrect php parameters in nginx host settings;
- an error directly in the php-script.

Your best helpers in troubleshooting any problem are logs:
```
/var/log/nginx/access.log  - лог запросов к nginx
/var/log/nginx/error.log   - лог ошибок nginx
/var/log/php7.2-fpm.log    - лог php-fpm
```
### Very long page load time (ERR_INCOMPLETE_CHUNKED_ENCODING)
Check for the `fastcgi_buffering off;` directive in the `location ~ \.php$` section of the site config.

### upstream sent too big header while reading response header from upstream
Raise the buffer size in the php section
```
fastcgi_buffers 16 16k; 
fastcgi_buffer_size 32k;
```
