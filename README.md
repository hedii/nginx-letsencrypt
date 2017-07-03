# nginx-letsencrypt

### Generate keys
```
sudo mkdir /etc/nginx/ssl
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 4096
sudo openssl rand -out /etc/nginx/ssl/ticket.key 48
```

### /etc/nginx/nginx.conf
```
user www-data;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    # ...
    
    gzip on;
    gzip_comp_level 5;
    gzip_vary on;
    gzip_types
      application/atom+xml
      application/javascript
      application/json
      application/rss+xml
      application/vnd.ms-fontobject
      application/x-font-ttf
      application/x-web-app-manifest+json
      application/xhtml+xml
      application/xml
      font/opentype
      image/svg+xml
      image/x-icon
      text/css
      text/plain
      text/x-component;
    
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### /etc/nginx/snippets/letsencrypt.conf
```
location ~ /.well-known {
    allow all;
}
```

### /etc/nginx/snippets/ssl.conf
```
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 208.67.222.222 208.67.220.220 216.146.35.35 216.146.36.36 valid=300s;
resolver_timeout 3s;
ssl_session_cache shared:SSL:100m;
ssl_session_timeout 24h;
ssl_session_tickets on;
ssl_session_ticket_key /etc/nginx/ssl/ticket.key;
ssl_dhparam /etc/nginx/ssl/dhparam.pem;
ssl_ecdh_curve secp384r1;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK';
```

### /etc/nginx/sites-enabled/example.com
```
server {
    listen 80;
    listen 443;
    server_name www.example.com;
    return 301 $scheme://example.com$request_uri;
}

server {
    listen 80;
    server_name example.com;
    root /home/debian/www/$host;
    include /etc/nginx/snippets/letsencrypt.conf;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name example.com;

    root /home/debian/www/$host;
    index index.html index.htm index.php;

    charset utf-8;
    sendfile off;
    client_max_body_size 100m;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    include /etc/nginx/snippets/ssl.conf;
    include /etc/nginx/snippets/letsencrypt.conf;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.1-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~* \.(woff|woff2|eot|ttf|jpg|jpeg|gif|css|png|js|ico)$ {
        access_log off;
        log_not_found off;
        expires max;
        gzip_vary on;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### certbot
```
sudo certbot certonly --webroot -w /home/debian/www/example.com -d example.com -d www.example.com

sudo certbot renew --dry-run
```

### /home/debian/reload_nginx.sh
```
#!/bin/bash
service nginx restart
```

### crontab
```
59 3 5 * * certbot renew --noninteractive --renew-hook /home/debian/reload_nginx.sh
```
