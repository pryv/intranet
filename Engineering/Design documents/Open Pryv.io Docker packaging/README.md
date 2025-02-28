Use cases



### native

1. dev: localhost - no NGINX, no SSL
2. dev: rec.la - no NGINX SSL
3. Prod: localhost + NGINX + your own SSL
4. Prod: localhost + some other SSL termination + its own SSL



### docker-compose

##### 1-Dev: builds everything, exposes core, without NGINX

Base for **3**

Goal: enthusiast, has developer environment, wants to customize assets & auth3

1. sh
2. git-clone: auth3, assets

docker-compose has no NGINX, binds 0.0.0.0:80 to core:9000

##### 2-Dev: builds everything, fetches rec.la cert, runs with NGINX

Goal: same as before, only has rec.la which is useful as some web apps require a https backend to work without tinkering

1. sh
2. git-clone: auth3, assets, rec.la

docker-compose file has NGINX with rec.la certs mounted in it

##### 3-Prod: already built images - no NGINX

Builds from **1**

Goal: download docker-compose file + config and launch it, you have your own SSL termination solution

Files:

- docker-compose
- Config core
- config email

##### 4-Prod: already built images - NGINX + certbot (SSL generation + auto-renew)

Goal: go to production in as little as possible

Files

- docker-compose
- config core
- config email
- config NGINX
