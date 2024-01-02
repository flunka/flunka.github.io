---
title:  'Getting Familiar With Nginx'
date:  2023-12-25T22:28:40+01:00
draft:  true
tags:
 - nginx
---
## Introduction
Some time ago I've stared working on web app in Go. I've been using standard `net/http` library to handle all requests. I have also implemented some basic logging and then I realized that I've written quite a bit code that not necessary is related with my app, but it does basic HTTP server stuff. So I've asked myself: "Do I need use some HTTP server like for Python web app? Or can I implement all with `net/http` Go library. I've started to looking for some answers and have found many statements that I don't need HTTP server and can implement everything in Go, but most of the times using HTTP sever like nginx or Apache can save me a lot of time and issues. Of course for learning purposes it is good idea to implement all by yourself. You will face many issues and rabbit holes by going this way and it will give you experience and understating, but if you want to deliver some app you can leverage existing solutions that solve all those problems.\
Keeping that in mind I've decided to learn more about nginx. I want to have better understating what it can provide and how I can leverage it. Currently I have some experience with nginx configuration, but I don't feel confident and I definitely not aware of all its features. The purpose of this post is to document my familiarizing with nginx.\
These notes are based on free book `NGINX Cookbook 2nd edition 2022` which is available on [Nginx site](https://www.nginx.com/resources/library/complete-nginx-cookbook/)
## Configuration
`/etc/nginx/nginx.conf` is default entry point. Directories `site-enabled` and `site-available` are deprecated. Directory `conf.d` should be used for config files.\
`nginx -t` - test configuration\
`nginx -s realod` - reload configuration
### Enable Stub Status
To enable stub status use `stub_status` directive in some location. It gives you information about:
* active connections
* total accepted connection
* connections handled
* requests served
This information is global. It is not specific to HTTP server where stub status is defined.
```
location /stub_status {
  stub_status;
  # Set IP restrictions as appropriate
  allow 127.0.0.1;
  deny all;
}
```
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
### HTTP/2
To configure HTTP/2 you have to use `http2` keyword in `listen` directive. It is possible to use HTTP/2 without SSL, but many clients do not support HTTP/2 without encryption. To use encryption to upstream use `grpcs://` protocol.

### HTTP/2 Example
```
upstream grpcservers {
  server backend1.local:50051;
  server backend2.local:50051;
}
server {
  listen 443 ssl http2 default_server;
  ssl_certificate server.crt;
  ssl_certificate_key server.key;
  location / {
    grpc_pass grpc://grpcservers;
  }
  location = /demo.html {
    http2_push /style.css;
    http2_push /image1.jpg;
  }
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
To limit HTTP connections you should create zone with `limit_conn_zone` directive (key is remote address in binary form, zone name is `limitbyaddr` and its size is 10MB). `limit_conn_status` specifies response code if limit is reached (503 is default). `limit_conn` directive takes two parameters: zone name and number of connections. You can use `limit_conn` in `http`, `server` and `location` context.
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
## Hardening
### Basic Authentication
Open source version of nginx only allows for basic HTTP authentication. Other methods are implemented in nginx plus.\
To enable HTTP basic authentication, you have to create file with credentials.
```
name1:password1
name2:password2:comment
name3:password3
```
Password have to be encrypted or hashed e.g. You can use openssl command to encrypt password: `openssl passwd YourPassword`
Next you can use `auth_basic` and `auth_basic_user_file` to enable HTTP basic authentication. It can be used in `http`, `server` and `location` context.
```
location / {
  auth_basic "Private site";
  auth_basic_user_file conf.d/passwd;
}
```


### Authentication Subrequest
In the book it is called authentication, but for me it looks like authorization mechanism.\
Open source nginx allows to authorize requests by third-party system. In order to do that you have to use `auth_reqest` with internal location. If you want to keep some data from subrequest response you can use `auth_request_set`.\
To remove body from subrequest use `proxy_pass_request_body` directive and change `Content-Length` header to `""`.\
If subrequest response status is 200, original request will be server.\
If subrequest response status is 401 or 403, original request will return the same status code.
```
location /private/ {
  auth_request /auth;
  auth_request_set $auth_status $upstream_status;
}
location = /auth {
  internal;
  proxy_pass http://auth-server;
  proxy_pass_request_body off;
  proxy_set_header Content-Length "";
  proxy_set_header X-Original-URI $request_uri;
}
```
### Limit by IP
You can limit access to specific IP addresses by using `allow` and `deny` directives. It can be used in `http`, `server`, `location` and `stream` context. It also can by combined with other mechanisms e.g. Basic authentication by using `satisfy` directive. With `satisfy any` only one criteria has to be met and with `satisfy all` all criteria have to be met.

```
location / {
  satisfy any;
  allow 192.168.1.0/24;
  deny all;
  auth_basic "closed site";
  auth_basic_user_file conf/htpasswd;
}
```

### HTTPS
To enable https you have to specify `ssl_certificate` and `ssl_certificate_key`.\
Other options also can be adjusted if needed.
```
http { # All directives used below are also valid in stream
  server {
    listen 8443 ssl;
    # Set accepted protocol and cipher
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    # RSA certificate chain loaded from file
    ssl_certificate /etc/nginx/ssl/example.crt;
    # RSA encryption key loaded from file
    ssl_certificate_key /etc/nginx/ssl/example.pem;
    # Elliptic curve cert from variable value
    ssl_certificate $ecdsa_cert;
    # Elliptic curve key as file path variable
    ssl_certificate_key data:$ecdsa_key_path;
    # Client-Server negotiation caching
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
  }
}
```
By default nginx does not verify upstream certificates. To enable verification `proxy_ssl_verify` directive can be used.
```
location / {
  proxy_pass https://upstream.example.com;
  proxy_ssl_verify on;
  proxy_ssl_verify_depth 2;
  proxy_ssl_protocols TLSv1.2;
}
```
If you want also listen for HTTP traffic but only serve https you can configure redirection.
```
server {
  listen 80 default_server;
  listen [::]:80 default_server;
  server_name _;
  return 301 https://$host$request_uri;
}
```
We can also instruct the browser to always send https requests by setting HSTS header. `add_header Strict-Transport-Security max-age=31536000;`
### Secure location
You can also limit access to resources by specifying some secret. Then to access that resources you have to generate link. To generate link you have to make MD5 hash with URI (`echo -n 'index.htmlmySecret' | openssl md5 -hex`) and your secret and then place hash before URI.  `www.example.com/resources/a53bee08a4bf0bbea978ddf736363a12/index.html`. The drawback of this method that hash in not user specific so anyone who get the link can access resource.
```
location /resources {
  secure_link_secret mySecret;
  if ($secure_link = "") { return 403; }
  rewrite ^ /secured/$secure_link;
}
location /secured/ {
  internal;
  root /var/www;
}
```
Another way secure access is to restrict access with expiration. This time we can specify any variable to create MD5 hash, so it can be user specific.\
```
location /resources {
  root /var/www;
  secure_link $arg_md5,$arg_expires;
  secure_link_md5 "$secure_link_expires$uri$remote_addrmySecret";
  if ($secure_link = "") { return 403; }
  if ($secure_link = "0") { return 410; }
}
```
To generate link we have to create MD5 hash again, but this time in slightly different way. It’s an MD5 hash in binary format, base64-encoded, with plus signs (+) translated to hyphens (-), slashes (/) translated to underscores (\_), and equal signs (=) removed.
```bash
echo -n '1924905600/resources/index.html127.0.0.1 mySecret' \
| openssl md5 -binary \
| openssl base64 \
| tr +/ -_ \
| tr -d =
```
MD5 hash and expiration date in the Unix epoch format we have to add in query string `/resources/index.html?md5=sqysOw5kMvQBL3j9ODCyoQ&expires=1924905600`

## Streaming
### Serving MP4 and FLV
The only thing to enable streaming is to use `mp4` or `flv` directive under desired location.
```
http {
  server {
    # ...
    location /videos/ {
      mp4;
    }
    location ~ \.flv$ {
      flv;
    }
  }
}
```
## Logs
### Logs format
To specify logs format use `log_format` directive (it is only valid in HTTP context). You have to specify name of format and how logs should looks like. You can define multiple logs formats. To use log format, use `access_log` directive with log file path and logs format name. `access_log` directive can be used in many context with different file path and format. Error logs can not have defined format. You can only specify file path and log level (default is error). Available log levels are `debug`, `info`, `notice`, `warn`, `error`, `crit`, `alert`, or `emerg`. Logs can be also forwarded to syslog server. To do that use `syslog` parameter and provide options:
* `server` (required) - IP, DNS or Unix socket
* `facility` (optional; default: `local7`) - one of 23 attributes defined in RFC for syslog
* `tag` (optional; default: `nginx`)
* `severity` (optional; default: `info`)
* `nohostname` (optional) - disables adding of the hostname filed into the syslog message header
```
http {
  log_format geoproxy
  '[$time_local] $remote_addr '
  '$realip_remote_addr $remote_user '
  '$proxy_protocol_server_addr $proxy_protocol_server_port '
  '$request_method $server_protocol '
  '$scheme $server_name $uri $status '
  '$request_time $body_bytes_sent '
  '$geoip_city_country_code3 $geoip_region '
  '"$geoip_city" $http_x_forwarded_for '
  '$upstream_status $upstream_response_time '
  '"$http_referer" "$http_user_agent"';
   server {
    access_log /var/log/nginx/access.log geoproxy;
    error_log /var/log/nginx/error.log warn;
  } 
  
  server {
    error_log syslog:server=10.0.1.42 debug;
    access_log syslog:server=10.0.1.42,tag=nginx,severity=info geoproxy;
  }
}
```
### Tracing
`$request_id` variable is available since nginx version `1.11.0`. It can be use to trace requests. It contains randomly generated string of 32 hex characters. In example below this variable is used in nginx logs format and it is also passed to upstream server (`proxy_set_header`) and it is added to response to client (`add_header`)
```
log_format trace '$remote_addr - $remote_user [$time_local] '
                  '"$request" $status $body_bytes_sent '
                  '"$http_referer" "$http_user_agent" '
                  '"$http_x_forwarded_for" $request_id';
upstream backend {
  server 10.0.0.42;
}
server {
  listen 80;
  # Add the header X-Request-ID to the response to the client
  add_header X-Request-ID $request_id;
  location / {
    proxy_pass http://backend;
    # Send the header X-Request-ID to the application
    proxy_set_header X-Request-ID $request_id;
    access_log /var/log/nginx/access_trace.log trace;
  }
}
```
