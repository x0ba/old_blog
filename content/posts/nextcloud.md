---
title: "Nextcloud"
date: 2023-04-11T22:20:08-07:00
draft: true
---

## Intro

Hey there. For a while now, I've been searching for a good cloud storage option. I've finally found it in Nextcloud. Self-hosting Nextcloud is no easy task, though. You *could* go with hosting Nextcloud with the Snap packaged version but nobody likes Snaps, and that means you would have to wait for someone to package the Snap for you every time a new version came out. You also could use Docker, but Docker networking hurts my brain, and adds an extra layer to jump through to bind Nextcloud to a domain. That's why I believe that installing from source is the best way to do things, as well as the most well documented way. I'm going to be assuming you already have a VPS running Ubuntu Server as well as a domain pointed to the IPV4 address of your VPS.

## Initial Steps

### Installing the Dependencies

A good installation of Nextcloud depends on three things: PHP, a database like MySQL, MariaDB or Postgre, and a web server. For the database, I chose to go with MariaDB, since it's easily the most performant out of the three. For the web server, I chose to go with Nginx, since that's what I'm the most familiar with even though most of the [Nextcloud docs](docs.nextcloud.com) are geared towards Apache. Install all these dependencies with 

```bash
apt install -y nginx python3-certbot-nginx mariadb-server php8.1 redis-server php8.1-{fpm,bcmath,bz2,intl,gd,mbstring,mysql,zip,xml,curl,imagick,opcache,apcu,redis}
```

You'll notice that we've installed quite a few PHP plugins, as well as a Redis server. These will come in handy later when we get into memory caching. For now, perform your first optimization by editing `/etc/php/8.1/fpm/pool.d/www.conf` and adding these lines:

```conf
pm = dynamic
pm.max_children = 120
pm.start_servers = 12
pm.min_spare_servers = 6
pm.max_spare_servers = 18
```

This increases the allowed number of child processes, which greatly speeds up the PHP side of our Nextcloud instance. Now start the MariaDB server by running

```bash
systemctl enable mariadb --now
```

### Setting up a SQL Database

Next we need to set up a SQL database, which Nextcloud will use to store all our user data. Run the following command:

```bash
mysql_secure_installation
```

Go ahead and run through the following questions, answering "yes" to all of them, and input a root password when prompted.

```
Switch to unix_socket authentication [Y/n]: Y
Change the root password? [Y/n]: Y	# Input a password.
Remove anonymous users? [Y/n]: Y
Disallow root login remotely? [Y/n]: Y
Remove test database and access to it? [Y/n]: Y
Reload privilege tables now? [Y/n]: Y
```

Now you should be able to sign into the SQL database with the root password you just set:

```bash
mysql -u root -p
```

Now we need to create a database for Nextcloud. Enter these commands one by one, replacing username and password with suitable values

```
CREATE DATABASE nextcloud;
GRANT ALL ON nextcloud.* TO 'username'@'localhost' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EXIT;
```

### HTTPS

Now we can configure HTTPS for our domain by obtaining an SSL certificate using Certbot.

```bash
certbot certonly --nginx -d nextcloud.example.org
```

Make sure to replace the example domain with the domain you have pointing at the IPV4 address of your server.

### Webserver Configuration

This Nginx configuration is taken straight from the Nextcloud docs. Make sure to replace any instances of `nextcloud.example.org` with your own domain and put this into `/etc/nginx/sites-available/nextcloud`.

```nginx

upstream php-handler {
    server unix:/var/run/php/php8.1-fpm.sock;
    server 127.0.0.1:9000;
}
map $arg_v $asset_immutable {
    "" "";
    default "immutable";
}
server {
    listen 80;
    listen [::]:80;
    server_name nextcloud.example.org ;
    return 301 https://$server_name$request_uri;
}
server {
    listen 443      ssl http2;
    listen [::]:443 ssl http2;
    server_name nextcloud.example.org ;
    root /var/www/nextcloud;
    ssl_certificate     /etc/letsencrypt/live/nextcloud.example.org/fullchain.pem ;
    ssl_certificate_key /etc/letsencrypt/live/nextcloud.example.org/privkey.pem ;
    client_max_body_size 512M;
    client_body_timeout 300s;
    fastcgi_buffers 64 4K;
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/wasm application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;
    client_body_buffer_size 512k;
    add_header Referrer-Policy                      "no-referrer"   always;
    add_header X-Content-Type-Options               "nosniff"       always;
    add_header X-Download-Options                   "noopen"        always;
    add_header X-Frame-Options                      "SAMEORIGIN"    always;
    add_header Strict-Transport-Security            "max-age=15552000; includeSubDomains"    always;
    add_header X-Permitted-Cross-Domain-Policies    "none"          always;
    add_header X-Robots-Tag                         "noindex, nofollow"          always;
    add_header X-XSS-Protection                     "1; mode=block" always;
    fastcgi_hide_header X-Powered-By;
    index index.php index.html /index.php$request_uri;
    location = / {
        if ( $http_user_agent ~ ^DavClnt ) {
            return 302 /remote.php/webdav/$is_args$args;
        }
    }
    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }
    location ^~ /.well-known {
        location = /.well-known/carddav { return 301 /remote.php/dav/; }
        location = /.well-known/caldav  { return 301 /remote.php/dav/; }
        location /.well-known/acme-challenge    { try_files $uri $uri/ =404; }
        location /.well-known/pki-validation    { try_files $uri $uri/ =404; }
        return 301 /index.php$request_uri;
    }
    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }
    location ~ \.php(?:$|/) {
        # Required for legacy support
        rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+|.+\/richdocumentscode\/proxy) /index.php$request_uri;
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;
        try_files $fastcgi_script_name =404;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS on;
        fastcgi_param modHeadersAvailable true;
        fastcgi_param front_controller_active true;
        fastcgi_pass php-handler;
        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
        fastcgi_max_temp_file_size 0;
    }
    location ~ \.(?:css|js|svg|gif|png|jpg|ico|wasm|tflite|map)$ {
        try_files $uri /index.php$request_uri;
        add_header Cache-Control "public, max-age=15778463, $asset_immutable";
        access_log off;     # Optional: Don't log access to assets
        location ~ \.wasm$ {
            default_type application/wasm;
        }
    }
    location ~ \.woff2?$ {
        try_files $uri /index.php$request_uri;
        expires 7d;
        access_log off;
    }
    location /remote {
        return 301 /remote.php$request_uri;
    }
    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }
}
```

Now enable the site using

```bash
ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
```

### Installing Nextcloud Itself

Now that we have all the dirty stuff out of the way, we can download the latest release tarball from Nextcloud's website and unzip it into the `/var/www` directory.

```bash
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
tar -xjf latest.tar.bz2 -C /var/www
chown -R www-data:www-data /var/www/nextcloud
chmod -R 755 /var/www/nextcloud
```

Now start PHP-fpm and reload Nginx

```bash
systemctl enable php8.1-fpm --now
systemctl reload nginx
```

You should now have a running Nextcloud instance when you navigate to your domain. Choose a secure admin username and password, leave the data folder at the default value, and enter the credentials for the database user you set earlier. For the database name, choose `nextcloud`. Leave localhost as is, and click finish. Congratulations! You now have your own Nextcloud server.

You can feel free to stop right here, but you'll eventually notice how sluggish Nextcloud is, and if you navigate to the admin panel you'll notice several warnings staring at you. Menacingly. In the following section, we'll rectify those warnings and speed up our instance dramatically.

## Optimizations

### Data Caching

The thing that you'll see speed up our Nextcloud instance the most is enabling data caching with php-opcache and redis. You should already have these installed. 

Edit the configuration file at `/etc/redis/redis.conf`.

* Search for `port` and change `port 6379` to `port 0`
* Search for and uncomment 
```
unixsocket /var/run/redis/redis.sock
```
and uncomment and change the line below it to 
```
unixsocketperm 770
```

Save and exit. Start the redis server with

```bash
sudo systemctl enable --now redis-server
```

and add the redis user to the www-data group with

```
sudo usermod -a -G redis www-data
```

Edit the Nextcloud config at `/var/www/nextcloud/config/config.php` and add this before the closing parentheses and semicolon.

```php
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'distributed' => '\\OC\\Memcache\\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'filelocking.enabled' => 'true',
  'redis' => 
  array (
    'host' => '/var/run/redis/redis-server.sock',
    'port' => 0,
    'timeout' => 0.0,
  ),
```
Now when you go to the admin page in Nextcloud, there should no longer be a ‘Security & Setup Warning' about memory caching not being enabled.








