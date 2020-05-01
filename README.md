Containerizing PHP,Apache, and MySQL
===================================

### Introduction
The project structure is as follows :

```
/php-apache-mysql/
├── apache
│   ├── apache_php.conf
│   └── Dockerfile
├── Cache
│   ├── cbc42e4c979f69f0_0
│   └── index-dir
│       └── the-real-index
├── docker-compose.yml
├── Network Persistent State
├── php
│   └── Dockerfile
├── public
│   ├── config.php
│   ├── create.php
│   ├── delete.php
│   ├── dump
│   │   └── dump.sql
│   ├── error.php
│   ├── index.php
│   ├── read.php
│   └── update.php
└── README.md
```

Once this structure is replicated or cloned(using "git clone https://github.com/yogesh174/docker-project") with these files, Docker and Docker compose is installed locally, you can simply run "docker-compose up" from the root of the project to run this project, and point your browser to http://localhost:8080 to see the project running.

### Docker Compose

Docker compose allows us to define the dependencies for the services, networks, volumes, etc as code.

#### docker-compose.yml
```
version: "3"
services:
  php:
    build: './php/'
    volumes:
      - ./public:/var/www/html/
  apache:
    build: './apache/'
    depends_on:
      - php
      - mysql
    ports:
      - "8080:80"
    volumes:
      - ./public:/var/www/html/
  mysql:
    image: mysql:5.7
    restart: always
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
      # Initially creates employees table
      - ./public/dump:/docker-entrypoint-initdb.d/
    environment:
      MYSQL_ROOT_PASSWORD: "passwd"
      MYSQL_DATABASE: "mydb"
      MYSQL_USER: "user1"
      MYSQL_PASSWORD: "passwd"
volumes:
    db_data:
```

Here we are decoupling Apache, PHP and Mysql by building them out into separate containers.We will be using the below Dockerfiles to decouple them : 

#### apache/Dockerfile
```
FROM httpd:2.4.33-alpine

RUN apk update; \
    apk upgrade;

COPY apache_php.conf /usr/local/apache2/conf/apache_php.conf
RUN echo "Include /usr/local/apache2/conf/apache_php.conf" \
    >> /usr/local/apache2/conf/httpd.conf
```

#### php/Dockerfile
```
FROM php:7.2.7-fpm-alpine3.7

RUN apk update; \
    apk upgrade;

RUN docker-php-ext-install mysqli
```

### Networking

We're going to have Apache proxy connections which require PHP rendering to port 9000 of our PHP container, and then have the PHP container serve those out as rendered HTML.

We need an apache vhost configuration file that is set up to proxy these requests for PHP files to the PHP container. So in the Dockerfile for Apache we have defined above, we add this file and then include it in the base httpd.conf file. This is for the proxying which allows us to decouple Apache and PHP. Here we called it apache_php.conf and we have the proxying modules defined as well as the VirtualHost.

#### apache/apache_php.conf
```
ServerName localhost

LoadModule deflate_module /usr/local/apache2/modules/mod_deflate.so
LoadModule proxy_module /usr/local/apache2/modules/mod_proxy.so
LoadModule proxy_fcgi_module /usr/local/apache2/modules/mod_proxy_fcgi.so

<VirtualHost *:80>
    # Proxy .php requests to port 9000 of the php-fpm container
    ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php:9000/var/www/html/$1
    DocumentRoot /var/www/html/
    <Directory /var/www/html/>
        DirectoryIndex index.php
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    # Send apache logs to stdout and stderr
    CustomLog /proc/self/fd/1 common
    ErrorLog /proc/self/fd/2
</VirtualHost>
```

### Volumes

Both the PHP and Apache containers have access to a "volume" that we define in the docker-compose.yml file which maps the public folder of our repository to the respective services for them to access. When we do this, we map a folder on the host filesystem (outside of the container context) to inside of the running containers.This allows us to edit the file outside of the container, yet have the container serve the updated PHP code out as soon as changes are saved.

And we also use "db_data" volume to store the contents of database, so that even if containers are terminated the data won't get deleted.

### PHP CRUD application

We'll use the following PHP application to demonstrate everything:

"dump.sql" creates a table 'employees' in the mydb database. 

"config.php" connects the php code with the 'employees' table of mydb database.

"index.php" creates the frontend grid which displays records from "employees" table.

"create.php" generates web form that is used to insert records in the employees table.

"read.php" retrieves the records from the employees table.

"update.php" updates records in employees table.

"delete.php" deletes records in employees table.

"error.php" will be displyed if a request is invalid.

