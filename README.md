# Wordpress website
### README for Setting Up WordPress with Docker, Docker-Compose, and NGINX on Amazon EC2

## Prerequisites
1. Amazon EC2 Instance
2. NGINX
3. Docker and Docker-Compose

## Steps to Follow

### 1. Launch EC2 Instance
   - Instance Type: t2.micro
   - Operating System: Amazon Linux Kernel 5.10

### 2. Install Docker and Docker-Compose
   Connect to your EC2 instance via SSH and run the following commands:
   
   sudo yum update -y
   sudo yum install docker -y
   sudo systemctl start docker
   sudo systemctl enable docker
   sudo usermod -aG docker ec2-user

   # Install Docker Compose
   curl -SL https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
   sudo ln -s /usr/local/bin/docker-compose /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose
   
### 3. Run the Containers Using Docker-Compose
   Create a directory and navigate into it:
   
   mkdir wordpress-docker
   cd wordpress-docker
   

   Create a `docker-compose.yaml` file with the following content:
   
---
version: '3'
services:

    wordpress:
        image: wordpress
        restart: always
        ports: 
            - 8080:80
        environment:
            WORDPRESS_DB_HOST: db
            WORDPRESS_DB_USER: user1
            WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_password
            WORDPRESS_DB_NAME: wpdb
        secrets:
            - db_password
        volumes:
            - wordpress:/var/www/html
            
    db:
        image: mysql:5.7.28
        ports:
            - "3306:3306"
        environment:
            MYSQL_DATABASE: wpdb
            MYSQL_USER: user1
            MYSQL_PASSWORD_FILE: /run/secrets/db_password
            MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
        secrets:
            - db_password
            - db_root_password
        volumes:
            - db:/var/lib/mysql
        networks:
            - default
            
    phpmyadmin:
        image: phpmyadmin/phpmyadmin
        links:
            - db:db
        ports:
            - 8282:80
        environment:
            MYSQL_USER: user1
            MYSQL_PASSWORD_FILE: /run/secrets/db_password
            MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
        secrets:
            - db_password
            - db_root_password
secrets:
    db_password:
        file: db_password.txt
    db_root_password:
        file: db_root_password.txt
        
volumes:
    wordpress:
    db:

   Run the following command to start the containers:

   docker-compose up -d

### 4. Install and Start NGINX
 
   sudo amazon-linux-extras install nginx1 -y
   sudo systemctl start nginx
   sudo systemctl enable nginx
 
### 5. Access WordPress Website
   Access the WordPress website using the public IP of your EC2 instance on port 8080:

   http://<PublicIP>:8080

   Set up your WordPress site name, username, and password.

### 6. Configure NGINX as a Reverse Proxy
   Stop NGINX before making configuration changes:
   
   sudo systemctl stop nginx

   Edit the NGINX configuration file:

   sudo vi /etc/nginx/nginx.conf

   Add the following location block inside the `server` block:
 
   location / {
       proxy_pass http://127.0.0.1:8080/wp-admin/;
   }
 
   Start NGINX:
  
   sudo systemctl start nginx
   
### 7. Access WordPress Website via the EC2 Instance IP
   You should now be able to access your WordPress website using the public IP of your EC2 instance:
 
   http://<PublicIP>
  
## Additional Notes
- Ensure that the security group associated with your EC2 instance allows traffic on ports 80 (HTTP) and 8080.
- Make sure Docker and Docker-Compose are correctly installed and running.

This setup guide helps you deploy WordPress with Docker and Docker-Compose on an EC2 instance and use NGINX as a reverse proxy to manage the site.
