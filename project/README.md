# WEB
Динамический веб
## Развертывание веб приложения
1. Варианты стенда:
* nginx + php-fpm (laravel/wordpress) + python (flask/django) + js (react/angular);
* nginx + java (tomcat/jetty/netty) + go + ruby;
* своя комбинация.
2. Реализации на выбор:
* на хостовой системе через конфиги в /etc;
* деплой через docker-compose.
# Описание, выполнение домашнего задания:
1. Для выполнения задания используется Docker-compose. 
* database из образа mysql:8.0 и вольюмом ./dbdata:/var/lib/mysql;
* wordpress из образа wordpress:6.0.1-php8.0-fpm-alpine и вольюмом ./wordpress:/var/www/html;
* nginx из образа nginx:1.22.0-alpine и вольюмами ./wordpress:/var/www/html ./nginx-conf:/etc/nginx/conf.d;
* node из образа node:16.13.2-alpine3.15 и вольюмом /node:/opt/server;
* app из собственного docker-образа.
2. Деплой осуществляется с помощью docker-compose:
```
version: '3'
services:
  postgres:
    image: postgres:10.5
    container_name: postgres
    restart: always
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - '5432:5432'
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

  wordpress:
    image: ntninja/wordpress-postgresql:latest
    container_name: wordpress
    restart: unless-stopped
    environment:
      WORDPRESS_DB_HOST: postgres
      WORDPRESS_DB_NAME: postgres
      WORDPRESS_DB_USER: postgres
      WORDPRESS_DB_PASSWORD: postgres
    volumes:
      - wordpress:/var/www/html
    networks:
      - app-network
    depends_on:
      - postgres

  app:
    build: ./python
    container_name: app
    restart: always
    env_file:
      - .env
    command:
      gunicorn --workers=2 --bind=0.0.0.0:8000 mysite.wsgi:application
    networks:
      - app-network

  node:
    image: node:16.13.2-alpine3.15
    container_name: node
    working_dir: /opt/server
    volumes:
      - ./node:/opt/server
    command: node test.js
    networks:
      - app-network

  nginx:
    image: nginx:1.15.12-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - 8083:8083
      - 8081:8081
      - 8082:8082
    volumes:
      - wordpress:/var/www/html  # изменено с ./wordpress на именованный том
      - ./nginx-conf:/etc/nginx/conf.d
    networks:
      - app-network
    depends_on:
      - wordpress
      - app
      - node

volumes:
  wordpress:   # добавлено объявление именованного тома
  dbdata:

networks:
  app-network:
    driver: bridge

```
3. Конфигурационные файлы NGINX:
```
server {
     listen 8083;
     listen [::]:8083;

     server_name example.com www.example.com;

     index index.php index.html index.htm;

     root /var/www/html;

     location ~ /.well-known/acme-challenge {
             allow all;
             root /var/www/html;
     }

     location / {
             try_files $uri $uri/ /index.php$is_args$args;
     }

     location ~ \.php$ {
             try_files $uri =404;
             fastcgi_split_path_info ^(.+\.php)(/.+)$;
             fastcgi_pass wordpress:9000;
             fastcgi_index index.php;
             include fastcgi_params;
             fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
             fastcgi_param PATH_INFO $fastcgi_path_info;
     }

     location ~ /\.ht {
             deny all;
     }

     location = /favicon.ico {
             log_not_found off; access_log off;
     }
     location = /robots.txt {
             log_not_found off; access_log off; allow all;
     }
     location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
             expires max;
             log_not_found off;
     }
}
```
* django.conf
```
upstream django {
 server app:8000;
}

server {
    listen 8081;
    listen [::]:8081;
    server_name localhost;
    location / {
 	   try_files $uri @proxy_to_app;	
    }

    location @proxy_to_app {
 	   proxy_pass http://django;
 	   proxy_http_version 1.1;
 	   proxy_set_header Upgrade $http_upgrade;
 	   proxy_set_header Connection "upgrade";
 	   proxy_redirect off;
 	   proxy_set_header Host $host;
 	   proxy_set_header X-Real-IP $remote_addr;
 	   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 	   proxy_set_header X-Forwarded-Host $server_name;
    }
}
```
* node.conf
```
server {
 listen 8082;
 listen [::]:8082;
 server_name localhost;
    location / {
 	   proxy_pass http://node:3000;
 	   proxy_http_version 1.1;
 	   proxy_set_header Upgrade $http_upgrade;
 	   proxy_set_header Connection "upgrade";
 	   proxy_redirect off;
 	   proxy_set_header Host $host;
 	   proxy_set_header X-Real-IP $remote_addr;
 	   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 	   proxy_set_header X-Forwarded-Host $server_name;  
    }
}
```
4. Запуск контейнеров:
```
docker-compose up -d
...
root@dz-28:~/project# docker-compose ps
  Name                 Command               State                                                                  Ports
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
app         gunicorn --workers=2 --bin ...   Up
nginx       nginx -g daemon off;             Up      80/tcp, 0.0.0.0:8081->8081/tcp,:::8081->8081/tcp, 0.0.0.0:8082->8082/tcp,:::8082->8082/tcp, 0.0.0.0:8083->8083/tcp,:::8083->8083/tcp
node        docker-entrypoint.sh node  ...   Up
postgres    docker-entrypoint.sh postgres    Up      0.0.0.0:5432->5432/tcp,:::5432->5432/tcp
wordpress   docker-entrypoint2.sh php-fpm    Up      9000/tcp

```
