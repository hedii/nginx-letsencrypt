# nginx-letsencrypt

### Generate keys
```
sudo mkdir /etc/nginx/ssl
sudo openssl dhparam -out /etc/nginx/ssl/dhparam.pem 4096
sudo openssl rand -out /etc/nginx/ssl/ticket.key 48
```

### /etc/nginx/site-available/example.com
```
server {
    server_name www.example.com;
    return 301 $scheme://example.com$request_uri;
}

server {
    listen 80;
    server_name example.com;

    root /home/debian/www/$host;
    index index.html index.htm index.php;

    charset utf-8;
    sendfile off;
    client_max_body_size 100m;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.1-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /.well-known {
        allow all;
    }

    location ~ /\.ht {
        deny all;
    }
}
```
