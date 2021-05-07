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
