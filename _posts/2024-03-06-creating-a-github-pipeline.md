---
layout: post
date: 2024-03-06 14:00:00 +0200
title: "Creating a Github pipeline"
categories: [Development]
tags: [php, laravel, github, pipeline, ci]
image: /assets/img/posts/laravel-performance.png
---

In my previous post, I wrote about [setting up a server for your PHP application](/posts/setting-up-a-server-for-your-php-application). We don't want to manually deploy our application every time we make a change, so let's automate this process using Github Actions and with a Laravel application as an example.

## What is Github Actions?

Github Actions is a CI/CD tool that allows you to automate your workflow directly in your Github repository. You can create custom workflows that run on specific triggers, such as pushing code, creating pull requests, or releasing a new version. With Github Actions, you can build, test, and deploy your code without leaving Github.

## Prerequisites

### Using SSH key authentication

To deploy your application to your server using Github Actions, you need to set up SSH key authentication. This allows Github Actions to connect to your server securely without entering a password.

1. Generate an SSH key pair on your local machine (don't use your own key, create a new one):

```bash
ssh-keygen -o -a 100 -t ed25519
```

2. Copy the content of your public key (`id_ed25519.pub`) to your server into the `~/.ssh/authorized_keys` file.
3. Add your private key as a secret in your Github repository. Go to your repository settings, then `Secrets and variables`, then `Actions`, and add a new secret with the name `SSH_PRIVATE_KEY` and the content of your private key `id_ed25519`.

### Deployment Server IP

Just as the private key, you need to add the IP of your server as a variable (not a secret) in your Github repository. Use the following `DEPLOYMENT_SERVER_IP` as your variable.

## Creating the Workflow

Create a new file in your repository under `.github/workflows/ci.yml` with the following content:

```yaml
name: CI

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  code-style:
    name: Code Style
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup PHP with PECL extension
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: dom, curl, libxml, mbstring, zip, pcntl, bcmath, intl, gd
      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress
      - name: Run Pint
        run: ./vendor/bin/pint --test

  validate-composer:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup PHP with PECL extension
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
      - name: Validate composer file
        run: composer validate --no-check-all --strict

  larastan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup PHP with PECL extension
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, intl, gd
      - name: Install Dependencies
        run: composer install --prefer-dist --no-progress
      - name: Run Larastan
        run: ./vendor/bin/phpstan analyse

  tests:
    runs-on: ubuntu-latest
    needs: [code-style, validate-composer, larastan]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
      - name: Setup PHP with PECL extension
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, intl, gd
      - name: Install PHP Dependencies
        run: composer install --prefer-dist --no-progress --optimize-autoloader
      - name: Install NPM Dependencies
        run: npm install
      - name: Compile Assets
        run: npm run build
      - name: Run PestPHP Tests
        run: vendor/bin/pest -p

  deploy:
    runs-on: ubuntu-latest
    needs: [tests]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: SSH Key Setup
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, bcmath, intl, gd, yaml, sqlite, pdo_sqlite
      - uses: actions/setup-node@v4
        with:
          node-version: 20.x
      - name: Install PHP Dependencies
        run: composer install --prefer-dist --no-progress --optimize-autoloader --no-dev
      - name: Install NPM Dependencies
        run: npm install
      - name: Compile Assets
        run: npm run build
      - name: SSH Commands
        run: |
          VERSION=$(/bin/date '+%Y%m%d%H%M')
          rsync -avz -e "ssh -o StrictHostKeyChecking=no" --exclude node_modules --delete ./ root@${{ vars.DEPLOYMENT_SERVER }}:/var/www/releases/$VERSION/
          ssh -o StrictHostKeyChecking=no root@${{ vars.DEPLOYMENT_SERVER }} "cd /var/www/releases/$VERSION && VERSION=$VERSION ./scripts/deploy.sh"
```

Remove or add jobs and services as needed. This workflow will run on every push to the `main` branch and will execute the following steps:

1. Code Style: Check the code style using Pint.
2. Validate Composer: Validate the `composer.json` file.
3. Larastan: Analyze the code using Larastan.
4. Tests: Run the tests using PestPHP.
5. Deploy: Deploy the application to the server, if previous jobs succeeded.

Instead of using the `main` branch, you can also use a tag to deploy a specific version of your application.

### Deployment Script

In the last step of the pipeline, the `deploy.sh` script is executed on the server. This script is responsible for executing tasks that are necessary for your application. Create a `scripts/deploy.sh` file in your repository with the following content:

```bash
#!/usr/bin/env bash
set -e

if [ -z "$VERSION" ]; then
    echo "VERSION is undefined.\n"
    exit 0
fi

ENVIRONMENT="production"
DIR_ROOT=/var/www
DIR_RELEASE="$DIR_ROOT/releases/$VERSION"
DIR_CURRENT="$DIR_ROOT/current"

cd "$DIR_RELEASE" || exit 0

# remove default storage dir
rm -rf "$DIR_RELEASE/storage"
# link .env file and storage
ln -s "$DIR_ROOT/.env" "$DIR_RELEASE/.env"
ln -s "$DIR_ROOT/storage" "$DIR_RELEASE/storage"

# put application in maintenance mode
cd "$DIR_CURRENT" && php artisan down

# stop all services -- comment out for OCTANE
systemctl stop nginx
systemctl stop php8.3-fpm

# fix possible permission issues
chown -R root: "$DIR_ROOT"
chown --from=root:root -R www-data: "$DIR_ROOT"
find /var/www -type f ! -name 'database.sqlite' -exec chmod 770 {} +
find /var/www -type d ! -name 'database.sqlite' -exec chmod 770 {} +

# start services -- comment out for OCTANE
systemctl restart nginx
systemctl restart php8.3-fpm

# prepare application
cd "$DIR_RELEASE"
php artisan migrate --force
php artisan storage:link
php artisan config:clear && php artisan config:cache
php artisan view:clear && php artisan view:cache
php artisan route:clear && php artisan route:cache
php artisan optimize

# prepare services
rm -f /etc/nginx/sites-available/default
ln -sf "$DIR_RELEASE/config/servers/nginx/with-php-fpm.conf" /etc/nginx/sites-available/default
#ln -sf "$DIR_RELEASE/config/servers/nginx/with-octane.conf" /etc/nginx/sites-available/default
rm -f /etc/supervisor/conf.d/default.conf
ln -sf "$DIR_RELEASE/config/servers/supervisor/${ENVIRONMENT}.conf" /etc/supervisor/conf.d/default.conf
rm /etc/php/8.3/fpm/pool.d/www.conf
ln -sf "$DIR_RELEASE/config/servers/php/www.conf" /etc/php/8.3/fpm/pool.d/www.conf

rm -f "$DIR_CURRENT" && ln -sf "$DIR_RELEASE" "$DIR_CURRENT"

chown --from=root:root -R www-data: "$DIR_ROOT"

supervisorctl reread && supervisorctl reload
service php8.3-fpm restart
nginx -s reload

cd "$DIR_CURRENT" && php artisan up
cd "$DIR_ROOT/releases" || exit 0
# keep some, but not all releases
ls -r | tail -n +3 | sudo xargs rm -rf --
```

First, we prepare the current release of the application, put the application in maintenance mode and stop all services, run migrations, publish assets, optimize the application, and restart the services. We then link the new release, disable maintenance mode and clean up old releases.

The downtime in general should be very minimal, but of course, depends on your machine and migrations.

### Conclusion

With this pipeline, you can automate the deployment of your Laravel application to your server using Github Actions. Your application will also be build entirely during that process, so we don't have to do it on the actual server and use precious resources. You can customize the pipeline to fit your specific needs and add additional steps as required. Have fun
