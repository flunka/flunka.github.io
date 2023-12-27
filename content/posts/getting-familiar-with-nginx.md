---
title:  'Getting Familiar With Nginx'
date:  2023-12-25T22:28:40+01:00
draft:  true
---
## Introduction
Some time ago I've stared working on web app in Go. I've been using standard `net/http` library to handle all requests. I have also implemented some basic logging and then I realized that I've written quite a bit code that not necessary is related with my app, but it does basic HTTP server stuff. So I've asked myself: "Do I need use some HTTP server like for Python web app? Or can I implement all with `net/http` Go library. I've started to looking for some answers and have found many statements that I don't need HTTP server and can implement everything in Go, but most of the times using HTTP sever like nginx or Apache can save me a lot of time and issues. Of course for learning purposes it is good idea to implement all by yourself. You will face many issues and rabbit holes by going this way and it will give you experience and understating, but if you want to deliver some app you can leverage existing solutions that solve all those problems.\
Keeping that in mind I've decided to learn more about nginx. I want to have better understating what it can provide and how I can leverage it. Currently I have some experience with nginx configuration, but I don't feel confident and I definitely not aware of all its features. The purpose of this post is to document my familiarizing with nginx.\
These notes are base on free book `NGINX Cookbook 2nd edition 2022` which is available on [Nginx site](https://www.nginx.com/resources/library/complete-nginx-cookbook/)
## Configuration
Directories `site-enabled` and `site-available` are deprecated. Directory `conf.d` should be used for config files.\
`nginx -t` - test configuration\
`nginx -s realod` - reload configuration
## Load balancer
Nginx can load balance HTTP, TCP and UDP traffic.\
### Load balancing algorithms
Load balancing algorithm are:
* round robin (default)
* least connections - `least_con`
* least time (only with nginx plus) - `least_time`
* generic hash - `hash`
* random - `random`
* IP hash (only for HTTP) - `ip_hash`
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

