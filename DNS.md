一、

```
upstream wordpress {
   server 10.0.0.201:80;
   server 10.0.0.202:80;
}
server {
   listen 80 ;
   listen 443 ssl;
   server_name www.wang.org;
   client_max_body_size 100m;
   ssl_certificate /apps/nginx/conf/conf.d/ssl/www.wang.org.pem;
   ssl_certificate_key /apps/nginx/conf/conf.d/ssl/www.wang.org.key;
    if ( $scheme = http) {
       return 301 https://$server_name$request_uri;
   }
   location / {
       proxy_pass http://wordpress;
       proxy_set_header Host  $http_host;
     }
}
```





```
fastcgi_cache_path /data/nginx/cache keys_zone=cache_zone:10m;
server {
   listen 80;
   server_name www.wang.org;
   root /data/wordpress;
   index index.php index.html;
   client_max_body_size 100m;
   location ~ \.php$|/php-status|/ping {
         root           /data/wordpress;
         fastcgi_pass   127.0.0.1:9000;
         #fastcgi_pass unix:/run/php/php8.1-fpm.sock;
         fastcgi_index index.php;
         fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;
         fastcgi_param HTTPS on;
         include       fastcgi_params;
         fastcgi_cache       cache_zone;
         fastcgi_cache_key   $uri;
         fastcgi_cache_valid 200 302 10m;
         fastcgi_cache_valid 404     1m;
     }
}
```

