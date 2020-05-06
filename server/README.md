## Server configs
Usually running Nginx + Cloudflare + Gunicorn

#### Reminders
`/etc/ssh/sshd_config`:
* PermitRootLogin no
* ClientAliveInterval 120

#### `/etc/systemd/system/site.service`
Enable site with `systemctl enable site.service`
```
[Unit]
Description=Gunicorn instance to serve site
After=network.target

[Service]
User=andrew
Group=www-data
WorkingDirectory=/var/www/site
Environment="PATH=/var/www/site/venv/bin"
ExecStart=/var/www/site/venv/bin/gunicorn --workers 3 --bind unix:api.sock -m 007 wsgi:app

[Install]
WantedBy=multi-user.target
```

#### `nginx.conf`
```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    # Basic Settings
    sendfile on; 
    tcp_nopush on; 
    tcp_nodelay on; 
    keepalive_timeout 65; 
    types_hash_max_size 2048;
    server_tokens off; # don't show server version
    client_max_body_size 50M; # max upload size

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging Settings
    log_format main '$http_cf_connecting_ip - $http_x_forwarded_for - [$time_local] - ' 
        '$request - $status - $body_bytes_sent - $http_referer - $http_user_agent';
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log;

    # Disabling GZIP with SSL http://breachattack.com/#howitworks
    gzip off;

    # Virtual Host Configs
    include /etc/nginx/sites/*;

    # Deny direct IP access
    include /etc/nginx/conf.d/cloudflare-allow.conf;
    deny all;
}
```

#### `sites/server.conf`
nginx site configs 
```
server {
  listen 80;
  listen [::]:80;

  server_name drew.hu www.drew.hu;

  return 301 https://drew.hu$request_uri;
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;

  include includes/ssl.conf;

  server_name www.drew.hu;

  return 301 https://drew.hu$request_uri;
}

server {
  listen 443 ssl http2 deferred;
  listen [::]:443 ssl http2 deferred;

  server_name drew.hu;

  charset utf-8;

  include includes/ssl.conf;
  include includes/protect-system-files.conf;

  root /var/www/html;

  index index.html;

  # For frontend
  location / {
    try_files $uri $uri/ /index.html =404;
  }

  # For backend deploying gunicorn
  location / { 
    include proxy_params;
    proxy_pass http://unix:/var/www/path/to/api.sock;
    #proxy_pass http://localhost:5000;
  }
```

#### `sites/no-default`
```
# Drop requests for unknown hosts
# 
# If no default server is defined, nginx will use the first found server.
# To prevent host header attacks, or other potential problems when an unknown
# servername is used in a request, it's recommended to drop the request
# returning 444 "no response".

server {
  listen [::]:80 default_server deferred;
  listen 80 default_server deferred;
  return 444;
}

server {
  listen [::]:443 ssl default_server;
  listen 443 ssl default_server;
  include includes/ssl.conf;
  return 444;
}
```

#### `includes/ssl.conf`
https://github.com/h5bp/server-configs-nginx
```
ssl on; 

ssl_protocols              TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;

ssl_ciphers                ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS;
ssl_prefer_server_ciphers  on;

ssl_session_cache    shared:SSL:10m; # a 1mb cache can hold about 4000 sessions, so we can hold 40000 sessions
ssl_session_timeout  24h;

keepalive_timeout 300s;

ssl_certificate      /etc/ssl/certs/site.pem;
ssl_certificate_key  /etc/ssl/private/site.key;

ssl_client_certificate /etc/nginx/certs/cloudflare.crt;
ssl_verify_client on;

add_header Referrer-Policy no-referrer;
add_header X-Frame-Options SAMEORIGIN;
add_header X-XSS-Protection "1; mode=block";
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

#### `includes/protect-system-files.conf`
```
# Prevent clients from accessing hidden files (starting with a dot)
# This is particularly important if you store .htpasswd files in the site hierarchy
# Access to `/.well-known/` is allowed.
# https://www.mnot.net/blog/2010/04/07/well-known
# https://tools.ietf.org/html/rfc5785
location ~* /\.(?!well-known\/) {
  deny all;
}

# Prevent clients from accessing to backup/config/source files
location ~* (?:\.(?:bak|conf|dist|fla|in[ci]|log|psd|sh|sql|sw[op])|~)$ {
  deny all;
}
```

#### `certs/cloudflare.crt`
Cloudflare authenticated origin pulls: https://support.cloudflare.com/hc/en-us/articles/204899617-Authenticated-Origin-Pulls

#### `conf.d/cloudflare-allow.conf`
Cloudflare IP ranges https://www.cloudflare.com/ips/


