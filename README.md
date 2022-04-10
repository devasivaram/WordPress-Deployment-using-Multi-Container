# WordPress-Deployment-using-Multi-Container
Installing wordpress in Nginx using Docker-compose

## Description

Here we will build a multi-container WordPress installation. Which includes a MySQL database, an Nginx web server, and WordPress containers. We will also add SSL to the domain by obtaining TLS/SSL certificates with Let’s Encrypt

## Prerequisites

 1. Need server with Ubuntu 18.04 or above.
 2. Need to install Docker, you can use this for installing [Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04) .
 3. Need to install Docker-compose, you can use this for installing [Docker-compose](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-ubuntu-18-04) .
 4. Purchase a domain name or point you domain name to server IP. You can use this to purchase new [domain_name](http://www.freenom.com/en/index.html).
 5. Need to point: 
   * An A record with example.com pointing to your server’s public IP address.
   * An A record with www.example.com pointing to your server’s public IP address.
   (Replace example.com with your domain name in this entire git)

## Procedure

# 1. To configure Nginx web server

First, create a project directory for your project setup called project and navigate to it:
~~~sh
mkdir project && cd project
~~~

Make a directory and open a file for nginx config
~~~sh
mkdir nginx-conf
vim nginx-conf/nginx.conf
~~~

> In this file, we add a server block with directives for our server name and document root, we also add location blocks to direct the Certbot, PHP and assests.
~~~
server {
        listen 80;
        listen [::]:80;

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
~~~

## 2. Add required Environmental variables 

~~~sh
vim .env
~~~

> Add
~~~
MYSQL_ROOT_PASSWORD=your_root_password
MYSQL_USER=your_wordpress_database_user
MYSQL_PASSWORD=your_wordpress_database_password
~~~

The .env file conatins sensitive information, so you will need to tell Git and Docker what files not to copy to your Git repositories and Docker images, for that we use .gitignore and .dockerignore files.

Now you will need to initaite git in the current working directory (project):
~~~
git init

vim .gitignore
vim .dockerignore
~~~
> Add line ".env" in both files (.gitignore & .dockerignore).

## 3. Let's create yml file for testing above configuration

~~~sh
vim docker-compose.yml
~~~

**Here we are using mysql:8 image with conatainer name "database" and we use wordpress image 5.1.1-fpm-alpine with container name "wordpress", this image have php-fpm enabled. Also, we are using nginx:1.15.12-alpine for webserver/Nginx image with container name "webserver". We define certbot after the webserver container. All of these containers are added in same network "wpnet".**

> Add these to the file
~~~
version: '3'

services:
  database:
    image: mysql:8.0
    container_name: database
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes: 
      - db-data:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - wpnet

  wordpress:
    depends_on: 
      - database
    image: wordpress:5.1.1-fpm-alpine
    container_name: wordpress
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - wp-data:/var/www/html
    networks:
      - wpnet

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    ports:
      - "80:80"
    volumes:
      - wp-data:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - ./logs/nginx:/var/log/nginx/
      - certbot:/etc/letsencrypt
    networks:
      - wpnet

  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot:/etc/letsencrypt
      - wp-data:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --staging -d example.com -d www.example.com

volumes:
  certbot:
  wp-data:
  db-data:

networks:
  wpnet:
    driver: bridge  
~~~

We’re using an alpine image — the 1.21.6-alpine Nginx image. This service definition also volumes:

 volumes: We're specifying both named volumes and bind mounts here:

   * ./nginx/:/etc/nginx/conf.d: This will bind mount the Nginx configuration directory on the host to the relevant directory on the container, ensuring that any             changes we make to files on the host will be reflected in the container.
   * ./logs/nginx:/var/log/nginx/ : This will bind mount the Nginx log directory on the host to the log directory on the container.
   * wp-data:/var/www/html/: This will mount our WordPress application code to the /var/www/html/ directory
   * certbot:/etc/letsencrypt/: This will mount the relevant Let’s Encrypt certificates and keys for our domain to the appropriate directory on the container.


## Check the above code works and generate SSL:
~~~sh
docker-compose config
docker-compose up -d
~~~

**You can now check that your certificates have been mounted to the webserver container with docker-compose exec:**

~~~sh
docker-compose exec webserver ls -l /etc/letsencrypt/live/
~~~
>Output

-rw-r--r--    1 root     root           740 Apr  8 05:52 README
drwxr-xr-x    2 root     root            93 Apr  8 06:13 example.com

So the request will be successful, you can edit the certbot service definition to remove the --staging flag and replcae it with --force-renewal flag, which will tell Certbot that you want to request a new certificate with the same domains as an existing certificate. Then you may run below command which will recreate the certbot container. Also include the --no-deps option to tell Compose that it can skip starting the webserver service, since it is already running:

~~~sh
docker-compose up --force-recreate --no-deps certbot
~~~

After you've installed your certificates, you can alter your Nginx settings to include SSL.

## 4. Modify Nginx conf and yml file

Adding an HTTP redirect to HTTPS, defining our SSL certificate and key locations, and adding security settings and headers are all part of enabling SSL in our Nginx configuration. Since you're going to recreate the webserver, stop the webserver container now.

~~~sh
docker-compose stop webserver
~~~

Before adding configuration to nginx config file, let’s first get the recommended Nginx security parameteres

~~~sh
curl -sSLo nginx-conf/options-ssl-nginx.conf https://raw.githubusercontent.com/certbot/certbot/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf
~~~

**Lets remove the old conf file and add new one

~~~
rm nginx-conf/nginx.conf
vim nginx-conf/nginx.conf
~~~

> Add this to the conf file
~~~
server {
        listen 80;
        listen [::]:80;

        server_name example.com www.example.com;

        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }

        location / {
                rewrite ^ https://$host$request_uri? permanent;
        }
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name example.com www.example.com;

        index index.php index.html index.htm;

        root /var/www/html;

        server_tokens off;

        ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

        include /etc/nginx/conf.d/options-ssl-nginx.conf;

        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src * data: 'unsafe-eval' 'unsafe-inline'" always;
        # add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        # enable strict transport security only if you understand the implications

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
~~~



## Modify yml file for describing the service definitions

~~~sh
vim docker-compose.yml
~~~

> Add these to the file:
~~~
version: '3'

services:
  database:
    image: mysql:8.0
    container_name: database
    env_file: .env
    environment:
      - MYSQL_DATABASE=wordpress
    volumes: 
      - db-data:/var/lib/mysql
    command: '--default-authentication-plugin=mysql_native_password'
    networks:
      - wpnet

  wordpress:
    depends_on: 
      - database
    image: wordpress:5.1.1-fpm-alpine
    container_name: wordpress
    env_file: .env
    environment:
      - WORDPRESS_DB_HOST=db:3306
      - WORDPRESS_DB_USER=$MYSQL_USER
      - WORDPRESS_DB_PASSWORD=$MYSQL_PASSWORD
      - WORDPRESS_DB_NAME=wordpress
    volumes:
      - wp-data:/var/www/html
    networks:
      - wpnet

  webserver:
    depends_on:
      - wordpress
    image: nginx:1.15.12-alpine
    container_name: webserver
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - wp-data:/var/www/html
      - ./nginx-conf:/etc/nginx/conf.d
      - ./logs/nginx:/var/log/nginx/
      - certbot:/etc/letsencrypt
    networks:
      - wpnet

  certbot:
    depends_on:
      - webserver
    image: certbot/certbot
    container_name: certbot
    volumes:
      - certbot:/etc/letsencrypt
      - wp-data:/var/www/html
    command: certonly --webroot --webroot-path=/var/www/html --email sammy@example.com --agree-tos --no-eff-email --force-renewal -d example.com -d www.example.com

volumes:
  certbot:
  wp-data:
  db-data:

networks:
  wpnet:
    driver: bridge
~~~

## Our setup is finished and let's recreate the container

~~~sh
docker-compose up -d --force-recreate --no-deps webserver
~~~

**Now, access the domain using the EC2 IP or EC2 hostname and set up the site:**
> ![image](https://user-images.githubusercontent.com/100773863/162557184-82a618d3-422b-4122-9eff-3a60327da29b.png)



## Conclusion
This is how we create wordpress site using docker-compose tool with Nginx and PHP. Please contact me when you encounter any difficulty error while using this terrform code. Thank you and have a great day!


### ⚙️ Connect with Me
<p align="center">
<a href="https://www.instagram.com/dev_anand__/"><img src="https://img.shields.io/badge/Instagram-E4405F?style=for-the-badge&logo=instagram&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/dev-anand-477898201/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a>


