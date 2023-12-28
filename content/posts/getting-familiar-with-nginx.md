---
title:  'Getting Familiar With Nginx'
date:  2023-12-25T22:28:40+01:00
draft:  true
---
## Introduction
Some time ago I've stared working on web app in Go. I've been using standard `net/http` library to handle all requests. I have also implemented some basic logging and then I realized that I've written quite a bit code that not necessary is related with my app, but it does basic HTTP server stuff. So I've asked myself: "Do I need use some HTTP server like for Python web app? Or can I implement all with `net/http` Go library. I've started to looking for some answers and have found many statements that I don't need HTTP server and can implement everything in Go, but most of the times using HTTP sever like nginx or Apache can save me a lot of time and issues. Of course for learning purposes it is good idea to implement all by yourself. You will face many issues and rabbit holes by going this way and it will give you experience and understating, but if you want to deliver some app you can leverage existing solutions that solve all those problems.\
Keeping that in mind I've decided to learn more about nginx. I want to have better understating what it can provide and how I can leverage it. Currently I have some experience with nginx configuration, but I don't feel confident and I definitely not aware of all its features. The purpose of this post is to document my familiarizing with nginx.\
These notes are based on free book `NGINX Cookbook 2nd edition 2022` which is available on [Nginx site](https://www.nginx.com/resources/library/complete-nginx-cookbook/)
## Configuration
Directories `site-enabled` and `site-available` are deprecated. Directory `conf.d` should be used for config files.\
`nginx -t` - test configuration\
`nginx -s realod` - reload configuration
## Load balancer
Nginx can load balance HTTP, TCP and UDP traffic.
### Load balancing algorithms
Load balancing algorithm are:
* round robin (default)
* least connections - `least_conn`
* least time (only with nginx plus) - `least_time`
* generic hash - `hash`
* random - `random`
* IP hash (only for HTTP) - `ip_hash`\

`backup` server will be used when both primary servers are down. The health of upstream server is passively checked by default by monitoring the response of client requests (`max_fails=1` and `fail_timeout=10s`, so if one request failed, server will be excluded from pool for 10 sec). Active health checks are available with nginx plus.
### HTTP
To configure load balancing for HTTP traffic you have to define server pool in `upstream` context and then redirect traffic to server pool by `proxy_pass` directive.
### HTTP example
```
upstream backend {
  server 10.10.12.45:80 weight=1 max_fails=3 fail_timeout=3s;
  server app.example.com:80 weight=2;
  server spare.example.com:80 backup;
}
server {
  location / {
    proxy_pass http://backend;
  }
}
```
### TCP/UDP
To configure load balancing for TCP/UDP traffic you have to define `stream` context and inside that context sever pool should be defined in `upstream` context. This configuration can not be added in `conf.d` directory because this directory is inside `http` block. You should create `stream.conf.d` directory and include that directory in main config file. 
### TCP/UDP example
Main config
```
stream {
  include /etc/nginx/stream.conf.d/*.conf;
}
```
stream.conf
```
upstream mysql_read {
  least_conn;
  server read1.example.com:3306;
  server read2.example.com:3306;
  server 10.10.12.34:3306 backup;
}
server {
  listen 3306;
  proxy_pass mysql_read;
}

upstream ntp {
  server ntp1.example.com:123 weight=2;
  server ntp2.example.com:123;
}
server {
  listen 123 udp;
  proxy_pass ntp;
}
```
## Traffic management
### A/B testing
To split request in specified proportion you can use `split_clients` module. It takes 3 parameters:
* string for hash
* output variable
* proportion mapping
### A/B testing example
This example splits requests to static file. 33.3% percent of users will get file from `sitev2` dir and rest of users from `sitev1` dir.
```
split_clients "${remote_addr}" $site_root_folder {
  33.3% "/var/www/sitev2/";
  * "/var/www/sitev1/";
}
server {
  listen 80 _;
  root $site_root_folder;
  location / {
    index index.html;
  }
}
```
### GeoIP
#### Installation
```bash
apt-get install nginx-module-geoip
mkdir /etc/nginx/geoip
cd /etc/nginx/geoip
wget "http://geolite.maxmind.com/download/geoip/database/GeoLiteCountry/GeoIP.dat.gz"
gunzip GeoIP.dat.gz
wget "http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz"
gunzip GeoLiteCity.dat.gz
```
#### Usage
```
load_module "/usr/lib64/nginx/modules/ngx_http_geoip_module.so";
http {
  geoip_country /etc/nginx/geoip/GeoIP.dat;
  geoip_city /etc/nginx/geoip/GeoLiteCity.dat;
  # ...
}
```
#### Variables
`geoip_country` and `geoip_city` give variables that start with `$geoip`.\
Those variables can be pass to your application or some traffic decisions can be made based on it.\
All variables are available at [nginx site](http://nginx.org/en/docs/varindex.html)

### Limiting connections
To limit HTTP connections you should create zone with `limit_conn_zone` directive (key is remote address in binary form, zone name is limitbyaddr and its size is 10MB). `limit_conn_status` specifies response code if limit is reached (503 is default). `limit_conn` directive takes two parameters: zone name and number of connections. You can use `limit_conn` in `http`, `server` and `location` context.
```
http {
  limit_conn_zone $binary_remote_addr zone=limitbyaddr:10m;
  limit_conn_status 429;
  # ...
  server {
    # ...
    limit_conn limitbyaddr 40;
    # ...
  }
}
```
### Limiting rate
To limit rate of HTTP requests you can create zone with `limit_req_zone` directive to set allowed request rate. To `limit_req` directive you can add two optional parameters `burst` and `delay`. `burst` will allow to exceed rate limit, but then all requests will be rejected. `delay` specifies how many packets can be made up front without throttling. It can be use in `http`(to verify), `server` and `location` contexts.
```
http {
  limit_req_zone $binary_remote_addr zone=limitbyaddr:10m rate=3r/s;
  limit_req_status 429;
  # ...
  server {
    # ...
    limit_req zone=limitbyaddr;
    location / {
      limit_req zone=limitbyaddr burst=12 delay=9;
    }
    # ...
  }
}
```
### Limiting bandwidth
You can limit bandwidth with `limit_rate_after` and `limit_rate` directives. Limiting bandwidth is per connection, so it connection limit should also be configured. It can be used in `http`, `server` and `location` context.
Following example limits bandwidth to 1 MB/sec after 10MB.

```
location /download/ {
  limit_rate_after 10m;
  limit_rate 1m;
}
```
## Caching
To create cache for requests you can use `proxy_cache_path` and specify:
* path to directory that will be created for cache,
* zone name for shared memory and its size (in example name is CACHE and size is 60MB)
* (optional) directory structure (`levels`)
* (optional) when release cached entries (`inactive`)
* (optional) max size of cache (`max_size`)

`proxy_cache_path` can be used only in `http` context.\
To use created cache you have to use `proxy_cache` directive in `http`, `server` or `location` context.\
You can define input for hash with `proxy_cache_key` directive.\
Cache can be bypassed by using `proxy_cache_bypass` directive. In example cache for location `/live` will be bypassed when request contains header `cache_bypass`.\
To increase nginx performance you can add header to response to allow client to cache responses. In example all css and js files should cached on client side and expired after one year. `~*` modifier after `location` means that URI should be match for case insensitive regular expression.
### Example
```
http {
  proxy_cache_path /var/nginx/cache keys_zone=CACHE:60m levels=1:2 inactive=3h max_size=20g;
  server {
    proxy_cache CACHE;
    proxy_cache_key "$host$request_uri $cookie_user";
    location /live {
        proxy_cache_bypass $http_cache_bypass;
    }
    location ~* \.(css|js)$ {
      expires 1y;
      add_header Cache-Control "public";
    }
  }
}
```
