# Renaming project to **Vouch** in January 2019

In the new year we will move the project to [vouch/vouch](https://github.com/vouch/vouch).  This is to [avoid a naming conflict](https://github.com/LassoProject/lasso/issues/35) with another project.

Other namespaces will be changed at the same time including the docker hub repo [lassoproject/lasso](https://hub.docker.com/r/lassoproject/lasso/) which will become [voucher/vouch](https://hub.docker.com/r/voucher/vouch)

Sorry for the inconvenience but we wanted to make this change at this relatively early stage of the project.

# Lasso

an SSO solution for nginx using the [auth_request](http://nginx.org/en/docs/http/ngx_http_auth_request_module.html) module.

lasso supports OAuth login via Google, [GitHub](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/about-authorization-options-for-oauth-apps/), [IndieAuth](https://indieauth.spec.indieweb.org/), and OpenID Connect providers

If lasso is running on the same host as the nginx reverse proxy the response time from the `/validate` endpoint to nginx should be less than 1ms

For support please file tickets here or visit our IRC channel [#lasso](irc://freenode.net/#lasso) on freenode

## Installation

* `cp ./config/config.yml_example ./config/config.yml`
* create OAuth credentials for lasso at [google](https://console.developers.google.com/apis/credentials) or [github](https://developer.github.com/apps/building-integrations/setting-up-and-registering-oauth-apps/about-authorization-options-for-oauth-apps/)
  * be sure to direct the callback URL to the `/auth` endpoint
* configure nginx...

```{.nginxconf}
server {
    listen 443 ssl http2;
    server_name dev.yourdomain.com;
    root /var/www/html/;

    ssl_certificate /etc/letsencrypt/live/dev.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dev.yourdomain.com/privkey.pem;

    # send all requests to the `/validate` endpoint for authorization
    auth_request /validate;

    location = /validate {
      # lasso can run behind the same nginx-revproxy
      # May need to add "internal", and comply to "upstream" server naming
      proxy_pass http://lasso.yourdomain.com:9090;

      # lasso only acts on the request headers
      proxy_pass_request_body off;
      proxy_set_header Content-Length "";

      # pass X-Lasso-User along with the request
      auth_request_set $auth_resp_x_lasso_user $upstream_http_x_lasso_user;

      # these return values are used by the @error401 call
      auth_request_set $auth_resp_jwt $upstream_http_x_lasso_jwt;
      auth_request_set $auth_resp_err $upstream_http_x_lasso_err;
      auth_request_set $auth_resp_failcount $upstream_http_x_lasso_failcount;
    }

    # if validate returns `401 not authorized` then forward the request to the error401block
    error_page 401 = @error401;

    location @error401 {
        # redirect to lasso for login
        return 302 https://lasso.yourdomain.com:9090/login?url=$scheme://$http_host$request_uri&lasso-failcount=$auth_resp_failcount&X-Lasso-Token=$auth_resp_jwt&error=$auth_resp_err;
    }

    # proxy pass authorized requests to your service
    location / {
      proxy_pass http://dev.yourdomain.com:8080;
      #  may need to set
      #    auth_request_set $auth_resp_x_lasso_user $upstream_http_x_lasso_user
      #  in this bock as per https://github.com/LassoProject/lasso/issues/26#issuecomment-425215810
      # set user header (usually an email)
      proxy_set_header X-Lasso-User $auth_resp_x_lasso_user;
    }
}

```

If lasso is configured behind the **same** nginx reverseproxy (perhaps so you can configure ssl) be sure to pass the `Host` header properly, otherwise the JWT cookie cannot be set into the domain

```{.nginxconf}
server {
    listen 80 default_server;
    server_name lasso.yourdomain.com;
    location / {
       proxy_set_header Host lasso.yourdomain.com;
       proxy_pass http://127.0.0.1:9090;
    }
}

```

## Running from Docker

```bash
docker run -d \
    -p 9090:9090 \
    --name lasso \
    -v ${PWD}/config:/config \
    -v ${PWD}/data:/data \
    lassoproject/lasso
```

The [lassoproject/lasso](https://hub.docker.com/r/lassoproject/lasso/) Docker image is an automated build on Docker Hub

[![docker-build status](https://img.shields.io/docker/build/lassoproject/lasso.svg)](https://hub.docker.com/r/lassoproject/lasso/builds/)

## Running from source

```bash
  go get ./...
  go build
  ./lasso
```

## the flow of login and authentication using Google Oauth

* Bob visits `https://private.oursites.com`
* the nginx reverse proxy...
  * recieves the request for private.oursites.com from Bob
  * uses the `auth_request` module configured for the `/validate` path
  * `/validate` is configured to `proxy_pass` requests to the authentication service at `https://lasso.oursites.com/validate`
    * if `/validate` returns...
      * 200 OK then SUCCESS allow Bob through
      * 401 NotAuthorized then
        * respond to Bob with a 302 redirect to `https://lasso.oursites.com/login?url=https://private.oursites.com`

* lasso `https://lasso.oursites.com/validate`
  * recieves the request for private.oursites.com from Bob via nginx `proxy_pass`
  * it looks for a cookie named "oursitesSSO" that contains a JWT
  * if the cookie is found, and the JWT is valid
    * returns 200 to nginx, which will allow access (bob notices nothing)
  * if the cookie is NOT found, or the JWT is NOT valid
    * return 401 NotAuthorized to nginx (which forwards the request on to login)

* Bob is first forwarded briefly to `https://lasso.oursites.com/login?url=https://private.oursites.com`
  * clears out the cookie named "oursitesSSO" if it exists
  * generates a nonce and stores it in session variable $STATE
  * stores the url `https://private.oursites.com` from the query string in session variable $requestedURL
  * respond to Bob with a 302 redirect to Google's OAuth Login form, including the $STATE nonce

* Bob logs into his Google account using Oauth
  * after successful login
  * Google responds to Bob with a 302 redirect to `https://lasso.oursites.com/auth?state=$STATE`

* Bob is forwarded to `https://lasso.oursites.com/auth?state=$STATE`
  * if the $STATE nonce from the url matches the session variable "state"
  * make a "third leg" request of google (server to server) to exchange the OAuth code for Bob's user info including email address bob@oursites.com
  * if the email address matches the domain oursites.com (it does)
    * create a user in our database with key bob@oursites.com
    * issue bob a JWT in the form of a cookie named "oursitesSSO"
    * retrieve the session variable $requestedURL and 302 redirect bob back to $requestedURL

Note that outside of some innocuos redirection, Bob only ever sees `https://private.oursites.com` and the Google Login screen in his browser.  While Lasso does interact with Bob's browser several times, it is just to set cookies, and if the 302 redirects work properly Bob will log in quickly.

Once the JWT is set, Bob will be authorized for all other sites which are configured to use `https://lasso.oursites.com/validate` from the `auth_request` nginx module.

The next time Bob is forwarded to google for login, since he has already authorized the lasso OAuth app, Google immediately forwards him back and sets the cookie and sends him on his merry way.  Bob may not even notice that he logged in via lasso.
