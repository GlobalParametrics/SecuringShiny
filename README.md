# Securing Shiny

### A summary of free/open source ways of securing Shiny
### [Thom Johnson](github.com/thomascjohnson)

[Shiny](https://shiny.rstudio.com/) is popular at Global Parametrics because it allows our scientists and economists to quickly visualize complex analysis without having to worry about setting up a full web application and without being limited by the feature sets of common data viz applications.

Shiny Server does not come with any built-in authentication options, but RStudio, the maker of Shiny, does offer a paid solution – [Shiny Server Pro](https://www.rstudio.com/products/shiny-server-pro/). I'm a huge fan of RStudio and everything they do. I've used Shiny Server Pro and it's a fine product. It has a lot of features, with one of the main ones being its range of authentication options. For startups they even offer a discount – which is great – but it's still a big price to pay for a startup or lone developer. It also still requires configuration and limits the number of users and the number environments/deployments of Shiny Server Pro to one per license. The methods in this guide are able to provide as many environments/deployments and as many users as you like – only bounded by operational costs.

Frankly, most people working with Shiny have no idea about authentication, security, software infrastructure and so forth, so it isn't unreasonable for them to fork over a few thousand dollars for a Shiny Server Pro license, or RStudio Connect, or [shinyapps.io](shinyapps.io). There's nothing wrong with that. However, with a little bit of time and effort you can save yourself that money, run as many environments and users as you wish, and get a much better grip on the engineering around serving web apps like the ones you make with shiny.

I'm writing this with the assumption that you are already using Shiny and you would like to know how you can securely add authentication (as in, username and password access and beyond) to your Shiny app. There are a variety of ways to secure a Shiny app, and most of them are difficult to the typical Shiny user. This review provides a set of Dockerfiles that serve as examples and instructions about how to set up each method. If you're unfamiliar with Docker, check out the [instructions for installing it](https://docs.docker.com/v17.12/install/) or see [this guide](https://docker-curriculum.com/).

## An Important Disclaimer

Before we start, you might be wondering the following:

Why can't you just write some login code that will be handled by Shiny? Well, first of all, [don't roll your own security](https://security.stackexchange.com/a/18198), and second of all, if you have trouble understanding how to implement security for Shiny in the first place, how will you be able to implement something that's actually secure? I've heard horror stories of Shiny apps that had some sort of security implemented in R/Shiny but ultimately exposed everything. I've even gone through the trouble of doing this myself for a client who had no other solution, and the result is an extremely painful system for authentication and authorizing users. When I was writing this article I browsed a few Shiny apps that implemented their own authentication in R, and the security was dubious.

Because Shiny isn't the right place to implement your security/authentication, you need to add some kind of layer between the user and the Shiny server. This is what's called a proxy. It will take requests to the server and check whether the user is authenticated. If the user is not authenticated, it will redirect them to an authentication page (as in a log in page, or something of the sort), and when the user is authenticated it will check whether the user is authorized, and if they are it will allow access to the Shiny application. The methods described here are all some variation on this idea.

## [Example App - Antipodes](github.com/GlobalParametrics/antipodes)

[This](github.com/GlobalParametrics/antipodes) is the example app we'll use. Run it:

```
docker pull globalparametrics/antipodes
docker run -d -p 8080:3838 \
  --name antipodescontainer \
  globalparametrics/antipodes
```

And see the app at [localhost:8080](http://localhost:8080)

Cleanup:

```
docker rm -f antipodescontainer
```

## [Simple HTTP Authentication](https://ipub.com/shiny-password-protect/)

### Pros:
* Extremely simple

### Cons:
* Doesn't scale
* Requires manual intervention in server config

The title to this section has a link to the methodology. I'm not going to cover it because it's extremely simple and extremely flawed – it doesn't scale.

## [Auth0 Proxy for Shiny](https://auth0.com/blog/adding-authentication-to-Shiny-server/)

### Pros:
* Not too hard to setup
* Auth0 is reasonably easy to use

### Cons:
* Locks you into Auth0
* Additional service/configuration to setup

This method is exclusive to using [Auth0](https://auth0.com/) as your identity provider, but if you were curious and wanted to use it with another identity provider (Google, your own Keycloak setup, or something else), you could make a few modifications to [this code](https://github.com/auth0/shiny-auth0) and it would work. If you are wondering how, get in touch with me.

The idea is the same – add a layer between Shiny and the user. In this case it's a reverse proxy written in Node JS using the [passport.js NPM](http://www.passportjs.org/). To run this example, you'll need to create an account on Auth0 (it's free) and create a domain and get the client id and client secret. Follow the instructions [found here in step 3](https://auth0.com/blog/adding-authentication-to-shiny-server/) to create an account, but skip the part about limiting logins to certain users. Do add http://localhost:8080/callback to your ```Allowed Callback URLs``` in the ```Client Settings``` page. With your credentials, write the following to a file named ```env```:

```
AUTH0_CLIENT_SECRET=<YOUR CLIENT SECRET>
AUTH0_CLIENT_ID=<YOUR CLIENT ID>
AUTH0_DOMAIN=<YOUR DOMAIN>
AUTH0_CALLBACK_URL=callback
COOKIE_SECRET=RandomValue314159
SHINY_HOST=localhost
SHINY_PORT=3838
PORT=8080
```

And replace ```<YOUR CLIENT SECRET>```, ```<YOUR CLIENT ID>``` and ```<YOUR DOMAIN>``` with the values from your Auth0 account.

Then pull the docker image:

```
docker pull globalparametrics/auth0
```

_Disclamer: this docker image is hacky in an attempt to get everything into one image. In the other examples I use networks. Forgive me, Docker gods, for creating a bad example._

and run it with the following, with the path to the ```env``` file you created in place of ```/path/to```:

```
docker run -d -p 8080:8080 \
  --name auth0container \
  -v /path/to/env:/shiny-auth0/env
  globalparametrics/auth0
```

And again go to [localhost:8080](http://localhost:8080/) to see the example.

Cleanup:

```
docker rm -f auth0container

# !! Only run this one if you want to delete the image too !!
docker rmi globalparametrics/auth0
```

## [Shinyproxy](https://www.shinyproxy.io/)

### Pros:
* Variety of authentication methods available
* Handles scaling Shiny apps
* Lots of features

### Cons:
* Difficult to setup in production
* Requires strong knowledge of Docker
* Issues that break Shiny functionality (does not update query strings due to use of iframes, for example)

Shinyproxy is a common choice for adding an authentication layer to your Shiny apps, while also adding options for scaling and application organization. The design of Shinyproxy revolves around encapsulating Shiny apps in Dockerfiles and therefore, docker images. Pull our Shinyproxy example image:

```
docker pull globalparametrics/shinyproxy
docker network create shinyproxy
```

and run it (switch the port 8080 to another if you're already running something on 8080):

```
docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --net shinyproxy \
  -p 8080:8080 \
  --name shinyproxycontainer \
  globalparametrics/shinyproxy
```

and access the example at the same url as above with the credentials:

```
username: user
password: pass
```

See the shinyproxy example here: [localhost:8080](http://localhost:8080/)

Cleanup the Shinyproxy containers and images with the following:

```
docker rm -f shinyproxycontainer
```

This example only makes use of the simple authentication option of Shinyproxy, but there are many other options that are far more secure and scalable, such as LDAP, Kerberos, OpenID Connect (including social providers, Auth0, Keycloak and more), SAML, and even custom authentication. Or of course, none at all, but that defeats the point of this guide. Check the options out [here](https://www.shinyproxy.io/configuration/#authentication).

Using the simple authentication is a **_really bad idea_** because just like the Simple HTTP Authentication method, it doesn't scale and even worse – it stores passwords in plaintext. You can try Shinyproxy with your Auth0 account.

### Using OpenID Connect and Auth0 with Shinyproxy

To do this, you will need to change your shinyproxy config. Follow these directions to do so.

In the config template below, fill in the `<AUTH_URL>`, `<TOKEN_URL>` and `<JWKS_URL>` with the `authorization_endpoint`, `token_endpoint` and `jwks_uri` found at the url made by replacing `<YOUR_DOMAIN>` with your Auth0 domain: `https://<YOUR_DOMAIN>.auth0.com/.well-known/openid-configuration`

Then replace `<CLIENT_ID>` and `<CLIENT_SECRET>` with the corresponding values for your Auth0 application.

```
proxy:
  port: 8080
  authentication: openid
  openid:
    auth-url: <AUTH_URL>
    token-url: <TOKEN_URL>
    jwks-url: <JWKS_URL>
    client-id: <CLIENT_ID>
    client-secret: <CLIENT_SECRET>
  docker:
      internal-networking: true
  specs:
  - id: antipodes
    display-name: Antipodes
    description: Find the opposite place on earth
    container-cmd: ["R", "-e", "antipodes::launchApp()"]
    container-image: globalparametrics/antipodes
    container-network: shinyproxy

logging:
  file:
    shinyproxy.log
```

Save this as `application.yml` and either navigate to the folder containing it or change the `$(pwd)` to the path to the folder containing it in the command below:

```
docker network create shinyproxy

docker run -d \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(pwd)/application.yml:/opt/shinyproxy/application.yml \
  --net shinyproxy -p 8080:8080 \
  --name shinyproxycontainer \
  globalparametrics/shinyproxy
```

See the shinyproxy example with login via Auth0 and OpenID Connect here: [localhost:8080](http://localhost:8080/)

```
docker rm -f shinyproxycontainer
docker rm -f $( \
  docker stop $( \
    docker ps -a -q --filter ancestor=globalparametrics/antipodes  \
    --format="{{.ID}}" \
  ) \
)

docker network rm shinyproxy

# !! Only run these if you want to delete the images too !!
docker rmi globalparametrics/shinyproxy
# This image is necessary for the other methods, so only delete if you don't
# want to try them
docker rmi globalparametrics/antipodes
```

## [Keycloak Gatekeeper Proxy](https://www.keycloak.org/docs/latest/securing_apps/index.html#_keycloak_generic_adapter)

### Pros:
* Lots of features, flexible

### Cons:
* Requires a Keycloak instance (or maybe not – it seems like you could use any OpenID Connect client but I didn't try)
* Hard to setup
* Additional service/configuration to setup


To use this authentication method, you'll need an instance of [Keycloak](https://www.keycloak.org/) running. In the example container this is already setup. This one requres a bit more setup:

### Pull the images:

```
docker pull jboss/keycloak
docker pull globalparametrics/keycloakgatekeeper
```

### Create a network for the containers:

```
docker network create keycloaknetwork
```

### Start the container keycloak container:

```
docker run -d \
  --network=keycloaknetwork \
  --name keycloakcontainer \
  --hostname keycloakcontainer \
  -e KEYCLOAK_USER=admin \
  -e KEYCLOAK_PASSWORD=pass \
  -p 8080:8080 \
  jboss/keycloak
```

### Create A Client

Login to keycloak at localhost:8080 withe the username `admin` and the password `pass` and click on the `Clients` button on the left, underneath `Realm Settings`.

Click the `Create` button at the top right and add the client ID "shiny". Switch the `Authorization Enabled` to `On` and add `http://localhost:8181/oauth/callback` to the `Valid Redirect URIs`. Click the save button and then go to `Credentials` and copy the `Secret` value.

Open a text editor and paste this secret value in place of the `<CLIENT_ID>` and save the file under the name `gatekeeper.conf` and remember the path to this file (you'll need it later):

```
client-id: shiny
client-secret: <CLIENT_ID>
discovery-url: http://keycloakcontainer:8080/auth/realms/master/.well-known/openid-configuration
enable-default-deny: true
listen: 0.0.0.0:8181
redirection-url: http://localhost:8181
upstream-url: http://127.0.0.1:3838
secure-cookie: false
resources:
- uri: /*
  methods:
  - GET
  roles:
  - admin
  require-any-role: true
- uri: /favicon
  white-listed: true
- uri: /css/*
  white-listed: true
- uri: /img/*
  white-listed: true
```

### Gatekeeper Workaround

Due to an issue with Keycloak Gatekeeper [explained here](https://issues.jboss.org/browse/KEYCLOAK-8954?focusedCommentId=13695636&page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel#comment-13695636), do the following:

1. Click the `Client Scopes` button on the left menu and click the `roles` link.
2. Click on the `Mappers` tab and click the `Create` button.
3. Put `aud-bug-workaround-script` in the name. Set the `Mapper Type` to `Script Mapper`.
4. Add this code in the script text area:
```    
token.addAudience(token.getIssuedFor());
token.getIssuer();
```
5. Set the `Token Claim Name` to `iss`, the `Claim JSON Type` to `String`, `Add to ID Token` to `On`, `Add to access token` to `On`, and `Add to userinfo` to `Off`. Then click save.

### Keycloak Container Host Mapping

In order for this to work, we need to one last thing. Run the following:

```
sudo cp /etc/hosts /etc/hosts.backup
sudo sh -c 'echo "127.0.0.1 keycloakcontainer" >> /etc/hosts'
```

### Try It Out

Start the container – use this as it is if the `gatekeeper.conf` file is in your working directory, otherwise either change your working directory to the one that contains it or switch the `$(pwd)` below to the path to the folder containing `gatekeeper.conf`:
```
docker run -d \
  -p 8181:8181 \
  --network=keycloaknetwork \
  --name gatekeepercontainer \
  --hostname gatekeepercontainer \
  -v $(pwd)/gatekeeper.conf:/gatekeeper.conf \
  globalparametrics/keycloakgatekeeper
```

And go to [localhost:8181](http://localhost:8181/). Log in with the admin/pass combination if it asks you, and you'll be shown the [antipodes](https://github.com/GlobalParametrics/antipodes) Shiny app.


### Cleanup

```
docker rm -f keycloakcontainer

docker rm -f gatekeepercontainer

docker network rm keycloaknetwork

sudo mv /etc/hosts.backup /etc/hosts

# !! Only run these if you want to delete the images too !!
docker rmi jboss/keycloak
docker rmi globalparametrics/keycloakgatekeeper
```

## [mod\_auth\_openidc](https://www.mod-auth-openidc.org/)

### Pros:
* Flexible, easy to setup if you're already using a reverse proxy

### Cons:
* You have to use apache, though if you really love nginx you could reverse proxy to nginx and then reverse proxy in nginx to shiny, but that sounds awful. Actually, there is an [nginx equivalent to this](https://github.com/tarachandverma/nginx-openidc).

This is my favorite way of adding authentication to Shiny, because it's the simplest and the most flexible. It adds OpenID Connect authentication, just like with Auth0 and the Keycloak Gatekeeper methods, but it is integreated into the Apache web server and allows substantial configuration without a lot of additional setup.

Before you start the containers, write the following to a file named `shiny.conf`, with:
1. Your Auth0 organization in place of `<ORGANIZATION>`
2. Your Auth0 client ID in place of `<CLIENT_ID>`
3. And your Auth0 client secret in place of `<CLIENT_SECRET`

```
# mods-available/auth_openidc.conf

Listen 8080

<VirtualHost *:8080>
  OIDCProviderMetadataURL https://<ORGANIZATION>.auth0.com/.well-known/openid-configuration
  OIDCClientID <CLIENT_ID>
  OIDCClientSecret <CLIENT_SECRET>

  OIDCScope "openid name email"
  OIDCRedirectURI http://localhost:8080/callback
  OIDCCryptoPassphrase SecuringShiny

  RewriteEngine on
  RewriteCond %{HTTP:Upgrade} =websocket
  RewriteRule /(.*) ws://antipodescontainer:3838/$1 [P,L]
  RewriteCond %{HTTP:Upgrade} !=websocket
  RewriteRule /(.*) http://antipodescontainer:3838/$1 [P,L]
  ProxyPass / http://antipodescontainer:3838/
  ProxyPassReverse / http://antipodescontainer:3838/
  ProxyRequests Off

  <Location />
     AuthType openid-connect
     Require valid-user
     LogLevel debug
  </Location>
</VirtualHost>
```

Now you're ready to try it out. Make sure to either replace the `$(pwd)` with the path to the `shiny.conf` file you created, or navigate the working directory in your terminal to the folder that contains `shiny.conf`.

```
docker pull globalparametrics/mod_auth_openidc

docker network create mod_auth_openidc
docker run -d -p 8080:8080 \
  --network=mod_auth_openidc \
  --name mod_auth_openidc_container \
  -v $(pwd)/shiny.conf:/etc/httpd/conf/sites-enabled/shiny.conf \
  globalparametrics/mod_auth_openidc

docker run -d \
  --network=mod_auth_openidc \
  --name antipodescontainer \
  --hostname antipodescontainer \
  globalparametrics/antipodes
```

Now go to [localhost:8080](http://localhost:8080). Just like in the Auth0 example, you'll be directed to log in, and when you do you'll see the Antipodes app.

Cleanup:

```
docker rm -f mod_auth_openidc_container
docker rm -f antipodescontainer
docker network rm mod_auth_openidc

# !! Only run these if you want to delete the images too !!
docker rmi globalparametrics/antipodes
docker rmi globalparametrics/mod_auth_openidc
```

## Summary

This is just a sample of the ways you can add authentication to Shiny without forking over a lot of money. You could modify the Auth0 proxy or write your own, for example. You could try this [nginx openidc module](https://github.com/tarachandverma/nginx-openidc).

By the way, the same logic here also applies to [RStudio Server](https://www.rstudio.com/products/rstudio/download-server/). You could apply the mod_auth_openidc method to that, or Keycloak Gatekeeper or even the Auth0 proxy.

With good standards, authentication is a lot easier and more secure. Get to know [Open ID](https://openid.net/), [SAML](https://en.wikipedia.org/wiki/Security_Assertion_Markup_Language) and [Kerberos](https://www.varonis.com/blog/kerberos-authentication-explained/).

If you have any questions about this guide, you can get in touch with me via my email in my [github profile](https://github.com/thomascjohnson).
