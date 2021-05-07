# Deploy Nuxt on Azure App Service

These are the steps on how to deploy the application on the azure portal(https://portal.azure.com/#main)

## Set Up Backend
Install nuxt-start

```bash
yarn add nuxt-start
```
Create a new folder called `server` in the root of the project. Then create an `index.js` file inside the `server` folder and paste the following inside the `index.js`:

```js

const {loadNuxt} = require('nuxt-start')
/*
  handle any uncaught exception that occurs during execution.
  Without this the server will stop on error
 */
process.on('uncaughtException', function (err) {
  // log the error on the console
  console.log(err)
})

async function start() {
  const nuxt = await loadNuxt('start')
  const port = process.env.PORT;
  const host = process.env.HOST;
  await nuxt.listen(port, host);
  console.log(`app listening on port ${port}`)
}

start().catch(
  e => console.log(e)
)

```
## Update Package.Json
Edit the start script to reflect the one below.
```json
{
  "scripts": {
    "start": "node server/index.js",
  }
}
```
## Update Nuxt.config.js
Then edit your nuxt.config.js to make sure there are no external file references.

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
## Add Web.config
On the root of your repository create a web.config file and add the code below.
```xml
<?xml version="1.0" encoding="utf-8"?>
<!--
     This configuration file is required if iisnode is used to run node processes behind
     IIS or IIS Express.  For more information, visit:

     https://github.com/tjanczuk/iisnode/blob/master/src/samples/configuration/web.config
-->

<configuration>
  <system.webServer>
    <!-- Visit https://azure.microsoft.com/en-us/blog/introduction-to-websockets-on-windows-azure-web-sites/ for more information on WebSocket support -->
    <webSocket enabled="false" />
    <handlers>
      <!-- Indicates that the server.js file is a Node.js site to be handled by the iisnode module -->
      <add name="iisnode" path="server" verb="*" modules="iisnode"/>
    </handlers>
    <rewrite>
      <rules>
        <!-- Do not interfere with requests for node-inspector debugging -->
        <rule name="NodeInspector" patternSyntax="ECMAScript" stopProcessing="true">
          <match url="^server\/debug[\/]?" />
        </rule>

        <!-- First we consider whether the incoming URL matches a physical file in the /public folder -->
        <rule name="StaticContent">
          <action type="Rewrite" url="public{REQUEST_URI}"/>
        </rule>

        <!-- All other URLs are mapped to the Node.js site entry point -->
        <rule name="DynamicContent">
          <conditions>
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="True"/>
          </conditions>
          <action type="Rewrite" url="server"/>
        </rule>
      </rules>
    </rewrite>

    <!-- 'bin' directory has no special meaning in Node.js and apps can be placed in it -->
    <security>
      <requestFiltering>
        <hiddenSegments>
          <remove segment="bin"/>
        </hiddenSegments>
      </requestFiltering>
    </security>

    <!-- Make sure error responses are left untouched -->
    <httpErrors existingResponse="PassThrough" />

    <!--
      You can control how Node is hosted within IIS using the following options:
        * watchedFiles: semi-colon separated list of files that will be watched for changes to restart the server
        * node_env: will be propagated to node as NODE_ENV environment variable
        * debuggingEnabled - controls whether the built-in debugger is enabled

      See https://github.com/tjanczuk/iisnode/blob/master/src/samples/configuration/web.config for a full list of options
    -->
    <!--<iisnode watchedFiles="web.config;*.js"/>-->
  </system.webServer>
</configuration>
```

## Create  A new App Service
Login to the azure portal on the home page you will see a create Service option just below the navbar.
After that you will be taken to  the app services page where on the menu below the navbar you will see an add button.
Click on it await to be redirected to the create app page.

###   Fill in Instance Details 
On the Instance section make sure to:-

* set publish as *code* **not** *docker* 
* set the run-time stack to Node 14 LTS. 
* choose Linux as Operating System.

### Connect to Github Repository
To connect the app to github you have to :-
* Enable continous deployment.
* Connect the github repository you want to deploy.

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

