# Deploy Nuxt on Azure App Service

These are the steps on how to deploy the application on the azure portal(https://portal.azure.com/)

## Set Up Backend

Create a new folder called `server` in the root of the project. Then create an `index.js` file inside the `server` folder and paste the following inside the `index.js`:

```js
const express = require('express')
const consola = require('consola')
const { loadNuxt } = require('nuxt-start')
const app = express()

async function start () {
  const nuxt = await loadNuxt(isDev ? 'dev' : 'start')
  await nuxt.listen(process.env.PORT, process.env.HOST)
}

start()

```

Then edit your nuxt.config.js to make sure there are no external file references:

Before:

```js
import pkg from './package'

export default {
... config
}
```

After:

```js
module.exports = {
... config
}

```
## Create  A new App Service
Login to the azure portal on the home page you will see a create Service option just below the navbar.
After that you will be taken to  the app services page where on the menu below the navbar you will see an add button.
Click on it await to be redirected to the create app page.

### Basic Information Section
On the INstance section make sure to set publish as code not docker ,set the run-time stack to Node 14 LTS and choose Linux as Operating System.

### Deployment Information Section
Enable continous deployment and connect the github repository you want to deploy.

### Set Environmental Variables

Make sure you set the following two environment variables (application settings) in App Service &rsaquo; Settings &rsaquo; Configuration &rsaquo; Application settings.

```
HOST: '0.0.0.0'
NODE_ENV: 'production'
```

# Nuxt NGINX Configuration

```nginx
map $sent_http_content_type $expires {
    "text/html"                 epoch;
    "text/html; charset=utf-8"  epoch;
    default                     off;
}

server {
    listen          80;             # the port nginx is listening on
    server_name     your-domain;    # setup your domain here

    gzip            on;
    gzip_types      text/plain application/xml text/css application/javascript;
    gzip_min_length 1000;

    location / {
        expires $expires;

        proxy_redirect                      off;
        proxy_set_header Host               $host;
        proxy_set_header X-Real-IP          $remote_addr;
        proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_read_timeout          1m;
        proxy_connect_timeout       1m;
        proxy_pass                          http://127.0.0.1:3000; # set the address of the Node.js instance here
    }
}
```

## Using nginx with generated pages and a caching proxy as fallback:

If you have a high volume website with regularly changing content, you might want to benefit from Nuxt generate capabilities and [nginx caching](https://www.nginx.com/blog/nginx-caching-guide).

Below is an example configuration. Keep in mind that:

- root folder should be the same as set by [configuration generate.dir](/docs/2.x/configuration-glossary/configuration-generate#dir)
- expire headers set by Nuxt are stripped (due to the cache)
- both Nuxt as nginx can set additional headers, it's advised to choose one (if in doubt, choose nginx)
- if your site is mostly static, increase the `proxy_cache_path inactive` and `proxy_cache_valid` numbers

If you don't generate your routes but still wish to benefit from nginx cache:

- remove the `root` entry
- change `location @proxy {` to `location / {`
- remove the other 2 `location` entries

```nginx
proxy_cache_path  /data/nginx/cache levels=1:2 keys_zone=nuxt-cache:25m max_size=1g inactive=60m use_temp_path=off;

map $sent_http_content_type $expires {
    "text/html"                 1h; # set this to your needs
    "text/html; charset=utf-8"  1h; # set this to your needs
    default                     7d; # set this to your needs
}

server {
    listen          80;             # the port nginx is listening on
    server_name     your-domain;    # setup your domain here

    gzip            on;
    gzip_types      text/plain application/xml text/css application/javascript;
    gzip_min_length 1000;

    charset utf-8;

    root /var/www/NUXT_PROJECT_PATH/dist;

    location ~* \.(?:ico|gif|jpe?g|png|woff2?|eot|otf|ttf|svg|js|css)$ {
        expires $expires;
        add_header Pragma public;
        add_header Cache-Control "public";

        try_files $uri $uri/ @proxy;
    }

    location / {
        expires $expires;
        add_header Content-Security-Policy "default-src 'self' 'unsafe-inline';";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        add_header X-Frame-Options "SAMEORIGIN";

        try_files $uri $uri/index.html @proxy; # for generate.subFolders: true
        # try_files $uri $uri.html @proxy; # for generate.subFolders: false
    }

    location @proxy {
        expires $expires;
        add_header Content-Security-Policy "default-src 'self' 'unsafe-inline';";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-Cache-Status $upstream_cache_status;

        proxy_redirect                      off;
        proxy_set_header Host               $host;
        proxy_set_header X-Real-IP          $remote_addr;
        proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_ignore_headers        Cache-Control;
        proxy_http_version          1.1;
        proxy_read_timeout          1m;
        proxy_connect_timeout       1m;
        proxy_pass                  http://127.0.0.1:3000; # set the address of the Node.js instance here
        proxy_cache                 nuxt-cache;
        proxy_cache_bypass          $arg_nocache; # probably better to change this
        proxy_cache_valid           200 302  60m; # set this to your needs
        proxy_cache_valid           404      1m;  # set this to your needs
        proxy_cache_lock            on;
        proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
        proxy_cache_key             $uri$is_args$args;
    }
}
```

## nginx configuration for Laravel Forge:

Change `YOUR_WEBSITE_FOLDER` to your website folder and `YOUR_WEBSITE_DOMAIN` to your website URL. Laravel Forge will have filled out these values for you but be sure to double check.

```nginx
# FORGE CONFIG (DOT NOT REMOVE!)
include forge-conf/YOUR_WEBSITE_FOLDER/before/*;

map $sent_http_content_type $expires {
    "text/html"                 epoch;
    "text/html; charset=utf-8"  epoch;
    default                     off;
}

server {
    listen 80;
    listen [::]:80;
    server_name YOUR_WEBSITE_DOMAIN;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    charset utf-8;

    gzip            on;
    gzip_types      text/plain application/xml text/css application/javascript;
    gzip_min_length 1000;

    # FORGE CONFIG (DOT NOT REMOVE!)
    include forge-conf/YOUR_WEBSITE_FOLDER/server/*;

    location / {
        expires $expires;

        proxy_redirect off;
        proxy_set_header Host               $host;
        proxy_set_header X-Real-IP          $remote_addr;
        proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_read_timeout          1m;
        proxy_connect_timeout       1m;
        proxy_pass                          http://127.0.0.1:3000; # set the address of the Node.js
    }

    access_log off;
    error_log  /var/log/nginx/YOUR_WEBSITE_FOLDER-error.log error;

    location ~ /\.(?!well-known).* {
        deny all;
    }
}

# FORGE CONFIG (DOT NOT REMOVE!)
include forge-conf/YOUR_WEBSITE_FOLDER/after/*;
```

## Secure Laravel Forge with TLS:

It's best to let Laravel Forge do the editing of the `nginx.conf` for you, by clicking on Sites -> YOUR_WEBSITE_DOMAIN (SERVER_NAME) and then click on SSL and install a certificate from one of the providers. Remember to activate the certificate. Your `nginx.conf` should now look something like this:

```nginx
# FORGE CONFIG (DOT NOT REMOVE!)
include forge-conf/YOUR_WEBSITE_FOLDER/before/*;

map $sent_http_content_type $expires {
    "text/html"                 epoch;
    "text/html; charset=utf-8"  epoch;
    default                     off;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name YOUR_WEBSITE_DOMAIN;

    # FORGE SSL (DO NOT REMOVE!)
    ssl_certificate /etc/nginx/ssl/YOUR_WEBSITE_FOLDER/258880/server.crt;
    ssl_certificate_key /etc/nginx/ssl/YOUR_WEBSITE_FOLDER/258880/server.key;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA:!3DES';
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/nginx/dhparams.pem;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    charset utf-8;

    gzip            on;
    gzip_types      text/plain application/xml text/css application/javascript;
    gzip_min_length 1000;

    # FORGE CONFIG (DOT NOT REMOVE!)
    include forge-conf/YOUR_WEBSITE_FOLDER/server/*;

    location / {
        expires $expires;

        proxy_set_header Host               $host;
        proxy_set_header X-Real-IP          $remote_addr;
        proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_redirect              off;
        proxy_read_timeout          1m;
        proxy_connect_timeout       1m;
        proxy_pass                          http://127.0.0.1:3000; # set the address of the Node.js
    }

    access_log off;
    error_log  /var/log/nginx/YOUR_WEBSITE_FOLDER-error.log error;

    location ~ /\.(?!well-known).* {
        deny all;
    }
}

# FORGE CONFIG (DOT NOT REMOVE!)
include forge-conf/YOUR_WEBSITE_FOLDER/after/*;
```

# PM2 Nuxt Deploy

Deploying using [PM2](https://pm2.keymetrics.io/) (Process Manager 2) is a fast and easy solution for hosting your universal Nuxt application on your server or VM.

In this guide, we build the application locally and serve it though a PM2 config file with the cluster mode activated. Cluster mode will prevent downtime by allowing applications to be scaled across multiple CPUs.

## Getting Started

Make sure you have pm2 installed on your server. If not, simply globally install it from yarn or npm.

```bash
# yarn pm2 install
$ sudo yarn global add pm2 --prefix /usr/local

# npm pm2 install
$ npm install pm2 -g
```

## Configure your application

All you need to add to your universal Nuxt app for serving it though PM2 is a file called `ecosystem.config.js`. Create a new file with that name in your root project directory and add the following content:

```javascript
module.exports = {
  apps: [
    {
      name: 'NuxtAppName',
      exec_mode: 'cluster',
      instances: 'max', // Or a number of instances
      script: './node_modules/nuxt/bin/nuxt.js',
      args: 'start'
    }
  ]
}
```

## Build and serve the app

Now build your app with `npm run build`.

And serve it with `pm2 start`.

Check the status `pm2 ls`.

Your Nuxt.js application is now serving!

## Further Information

This solution guarantees no downtime for your application on this server. (You should also prevent server failure through redundancy or high availability cloud solutions.)

You can find PM2 additional configurations [here](https://pm2.keymetrics.io/docs/usage/application-declaration/#general).

