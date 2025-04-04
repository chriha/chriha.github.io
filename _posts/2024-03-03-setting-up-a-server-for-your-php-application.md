---
layout: post
date: 2024-03-03 14:00:00 +0200
title: "Setting up a server for your PHP application"
categories: [Development]
tags: [php, admin, server, ngnix, sqlite, fail2ban, laravel, octane, fpm]
image: /assets/img/posts/server-setup.jpg
---

In this part, we'll discover how to set up a server for your PHP application. We will use a minimal setup with the current version of PHP, Nginx, and SQLite. We will also see how to configure the server to run a Laravel application.

Keep in mind, this guide is for educational purposes and should not be used in a production environment without proper security measures. While we will cover some basic security configurations, it's essential to consult with a security expert to ensure your server is secure.

## Creating a server

First, we need to create a server. I personally like my VPS (virtual private server) from DigitalOcean. You can use any other provider like AWS, Google Cloud, Hetzner or whatever you wish to use.

### What to consider when creating a server

- Make sure to choose the latest Ubuntu LTS version.
- Choose a server with at least 1GB of RAM. This is the minimum requirement for most PHP applications.
- Select a data center location that is closest to your target audience (latency matters).
- **Provide the public SSH key to access the server. If you don't have one, you can generate it using the `ssh-keygen` command.** Usually, the providers will create a root user with the SSH key you provide.

## Setting up the server

### SSH into the server

Once the server is created, you can SSH into the server:

```bash
ssh root@your_server_ip
```

This requires the SSH key you provided when creating the server. If you used a password, you will be prompted to enter it. I highly recommend using SSH keys though.

You can also add this to your `~/.ssh/config` file:

```bash
Host my-awesome-server
  HostName your_server_ip
  User root
  IdentityFile ~/.ssh/your_private_key
```

This way, you can simply run `ssh my-awesome-server` to connect to your server.

### fail2ban

First, let's install `fail2ban` to protect our server from brute-force attacks. `fail2ban` scans log files and bans IPs that show malicious signs, such as too many password failures.

```bash
apt update
apt install fail2ban
# check the status
systemctl status fail2ban
```

#### Configure fail2ban

Create a new configuration file for `fail2ban`:

```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Feel free to edit the configuration file to suit your needs. For example, you can set the `maxretry` value to 1 to ban the IP after the first failed attempt. Don't forget to restart the `fail2ban` service after making changes:

```bash
systemctl restart fail2ban
```

### Install dependencies

Now, let's install some dependencies:

```bash
apt install -y software-properties-common vim
```

### Install PHP, Nginx and other dependencies

```bash
# add additional extensions if needed
apt install -y curl php8.3 php8.3-cli php8.3-fpm php8.3-mbstring php8.3-xml \
    php8.3-curl php8.3-zip php8.3-bcmath php8.3-gd php8.3-intl php8.3-imagick \
    php8.3-sqlite3 nginx
# need supervisor?
apt install -y supervisor
```

#### Configure Nginx

Make a backup of the default Nginx configuration:

```bash
cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak
```

You can use a configuration like this for your Nginx server.

##### For PHP FPM

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/current/public;

    index index.php index.html index.htm;

    # # # # # # # # # # # # # # # # # # # # #
    # ERRORS
    error_page 404 /index.php;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    # # # # # # # # # # # # # # # # # # # # #
    # HEADERS
    add_header 'Access-Control-Allow-Origin' "$http_origin" always;
    add_header 'Access-Control-Allow-Credentials' 'true' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PATCH, DELETE, OPTIONS' always;
    add_header 'Access-Control-Allow-Headers' 'Content-Type, Accept, x-xsrf-token' always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header 'X-Content-Type-Options' nosniff always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    # recommendations
    # CSP is set by Laravel
    #add_header Content-Security-Policy "default-src 'self'; style-src 'self' fonts.googleapis.com; font-src 'self' fonts.gstatic.com; script-src 'self' 'unsafe-eval'; object-src 'none'; frame-ancestors 'self'; base-uri 'self'; form-action 'self'" always;
    add_header Permissions-Policy "geolocation=(self), microphone=(), camera=()" always;


    # # # # # # # # # # # # # # # # # # # # #
    # SECURITY
    # prevent download of the web.config
    location ~ /web\.config { return 404; }
    # don't show server information
    server_tokens off;

    # # # # # # # # # # # # # # # # # # # # #
    # APPLICATION
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    charset utf-8;
    client_max_body_size 1M;

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
    }

    location ~*  \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 365d;
    }

    gzip on;
    gzip_types text/plain text/css text/xml text/javascript application/json application/x-javascript application/xml application/xml+rss application/x-font-ttf font/opentype image/svg+xml image/x-icon;
}
```

##### For Laravel Octane

```nginx
# here we're using nginx as a proxy to forward incoming requests on port 80 to octane's 8080
upstream octane_upstream {
  ip_hash; # session persistance
  server 127.0.0.1:8080;
  keepalive 64;
}

server {
    listen 80;
    server_name _;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    location / {
	    proxy_redirect off;
        proxy_pass http://octane_upstream;
        proxy_http_version 1.1;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-NginX-Proxy    true;
        proxy_set_header Connection '';
    }

    # Handling static files
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|svg|css|js)$ {
        root /var/www/current/public;
        try_files $uri $uri/ =404;
        expires 365d;
    }

    # # # # # # # # # # # # # # # # # # # # #
    # SECURITY
    # prevent download of the web.config
    location ~ /web\.config { return 404; }
    # don't show server information
    server_tokens off;

    charset utf-8;
    client_max_body_size 1M;

    gzip on;
    gzip_types text/plain text/css text/xml text/javascript application/json application/x-javascript application/xml application/xml+rss application/x-font-ttf font/opentype image/svg+xml image/x-icon;
}
```

#### Prepare directories

Of course, we need directories in which our application can live. Here's a structure for a Laravel application:

```bash
mkdir -p /var/www/storage/app \
    /var/www/storage/framework/cache \
    /var/www/storage/framework/session \
    /var/www/storage/framework/views \
    /var/www/storage/logs \
    /var/www/releases
```

#### Add a SQLite database

```bash
touch /var/www/storage/database.sqlite

# make sure the www-data user can read and write to the database
chown www-data: /var/www/storage/database.sqlite
chmod 660 /var/www/storage/database.sqlite
```

#### Correct permissions

```bash
chown -R www-data: /var/www
find /var/www -type f ! -name 'database.sqlite' -exec chmod 770 {} +
find /var/www -type d ! -name 'database.sqlite' -exec chmod 770 {} +
```


### Install and configure cronjob

```bash
apt install -y cron
# add a cronjob for the www-data user
crontab -u www-data -l | { cat; echo "* * * * * php /var/www/current/artisan schedule:run >> /dev/null 2>&1"; } | crontab -u www-data -
```

### Configure Supervisor

If you need to run a queue worker or any other process, you can use Supervisor. Here is an example configuration for Supervisor:

```ini
# keep octane running
#[program:octane]
##process_name=%(program_name)s_%(process_num)02d
#process_name=%(program_name)s
#command=php /var/www/current/artisan octane:start --port=8080
#autostart=true
#autorestart=true
#stopasgroup=true
#killasgroup=true
#user=www-data
#numprocs=1
#redirect_stderr=true
#stopwaitsecs=3600
#stdout_logfile=/var/www/storage/logs/octane.log
#stdout_logfile_maxbytes=10MB
#stdout_logfile_backups=10

# keep the queue listener running
[program:queue-listener]
process_name=%(program_name)s
command=php /var/www/current/artisan queue:listen
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/storage/logs/worker.log
stopwaitsecs=3600

# if you're using horizon, you can use this program instead
#[program:horizon]
#process_name=%(program_name)s
#command=php /var/www/current/artisan horizon
#autostart=true
#autorestart=true
#stopasgroup=true
#killasgroup=true
#user=www-data
#numprocs=1
#redirect_stderr=true
#stdout_logfile=/var/www/storage/logs/worker.log
#stopwaitsecs=3600
```

### Restart services

```bash
systemctl restart nginx
systemctl restart php8.3-fpm
systemctl restart supervisor
```

### Add environment file

If you're using Laravel, you can add an environment file to `/var/www/.env`. During the deployment process, which we'll cover in the next post, you can symlink this file to the current release.

```bash

## Useful commands

Here are some useful commands to manage the server:

```bash
# Show current logged in users
who
# fail2ban jail status / banned IPs
fail2ban-client status sshd
```

## Conclusion and home assignment

Setting up a server for your PHP application is a crucial step in the development process. By following the steps provided, you can create a secure and efficient environment for your application to run.

For your home assignment, since we used the `root` user to set up the server, I recommend creating a new user with limited permissions.

In the [next post](/posts/creating-a-github-pipeline), we'll set up a GitHub pipeline for a Laravel application. The pipeline runs tests and deploys the application to our server.
