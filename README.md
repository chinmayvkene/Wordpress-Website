---

# WordPress Website Setup Guide

### Prerequisites
1. Amazon EC2 Instance
2. NGINX
3. Docker and Docker-Compose

### Steps to Follow

#### 1. Launch EC2 Instance
- **Instance Type:** t2.micro
- **Operating System:** Amazon Linux Kernel 5.10

#### 2. Install Docker and Docker-Compose
Connect to your EC2 instance via SSH and run the following commands:

```sh
sudo yum update -y
sudo yum install docker -y
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# Install Docker Compose
curl -SL https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
sudo chmod -R 777 /usr/local/bin/docker-compose /usr/bin/docker-compose
```

#### 3. Run the Containers Using Docker-Compose
Create a directory and navigate into it:

```sh
mkdir wordpress-docker
cd wordpress-docker
```

Create a `docker-compose.yaml` file with the following content:

```yaml
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

secrets:
  db_password:
    file: db_password.txt
  db_root_password:
    file: db_root_password.txt

volumes:
  wordpress:
  db:
```

Run the following command to start the containers:

```sh
docker-compose up -d
```

#### 4. Install and Start NGINX

```sh
sudo amazon-linux-extras install nginx1 -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

#### 5. Access WordPress Website
Access the WordPress website using the public IP of your EC2 instance on port 8080:

```
http://<PublicIP>:8080
```

Set up your WordPress site name, username, and password.

#### 6. Configure NGINX as a Reverse Proxy
Stop NGINX before making configuration changes:

```sh
sudo systemctl stop nginx
```

Edit the NGINX configuration file:

```sh
sudo vi /etc/nginx/nginx.conf
```

Add the following location block inside the `server` block:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080/wp-admin/;
}
```

Start NGINX:

```sh
sudo systemctl start nginx
```

#### 7. Access WordPress Website via the EC2 Instance IP
You should now be able to access your WordPress website using the public IP of your EC2 instance:

```
http://<PublicIP>
```

#### 8. Script for Log Report
Create a script for generating log reports:

```sh
cd /root/aws
vi script.sh
```

Add the following content to `script.sh`:

```bash
#!/bin/bash

# variables
LOGFILE="/var/log/nginx/access.log"
LOGFILE_GZ="/var/log/nginx/access.log.*"
RESPONSE_CODE="200"

# functions
filters(){
grep $RESPONSE_CODE \
| grep -v "\/rss\/" \
| grep -v robots.txt \
| grep -v "\.css" \
| grep -v "\.jss*" \
| grep -v "\.png" \
| grep -v "\.ico"
}

filters_404(){
grep "404"
}

request_ips(){
awk '{print $1}'
}

request_method(){
awk '{print $6}' \
| cut -d'"' -f2
}

request_pages(){
awk '{print $7}'
}

wordcount(){
sort \
| uniq -c
}

sort_desc(){
sort -rn
}

return_kv(){
awk '{print $1, $2}'
}

return_top_ten(){
head -10
}

## actions
get_request_ips(){
echo ""
echo "Top 10 Request IP's:"
echo "===================="

cat $LOGFILE \
| filters \
| request_ips \
| wordcount \
| sort_desc \
| return_kv \
| return_top_ten
echo ""
}

get_request_methods(){
echo "Top Request Methods:"
echo "===================="
cat $LOGFILE \
| filters \
| request_method \
| wordcount \
| return_kv
echo ""
}

get_request_pages_404(){
echo "Top 10: 404 Page Responses:"
echo "==========================="
zgrep '-' $LOGFILE $LOGFILE_GZ \
| filters_404 \
| request_pages \
| wordcount \
| sort_desc \
| return_kv \
| return_top_ten
echo ""
}

get_request_pages(){
echo "Top 10 Request Pages:"
echo "====================="
cat $LOGFILE \
| filters \
| request_pages \
| wordcount \
| sort_desc \
| return_kv \
| return_top_ten
echo ""
}

get_request_pages_all(){
echo "Top 10 Request Pages from All Logs:"
echo "==================================="
zgrep '-' --no-filename $LOGFILE $LOGFILE_GZ \
| filters \
| request_pages \
| wordcount \
| sort_desc \
| return_kv \
| return_top_ten
echo ""
}

# executing
get_request_ips
get_request_methods
get_request_pages
get_request_pages_all
get_request_pages_404
```

Make the script executable and run it:

```sh
chmod -R 777 script.sh
sh /root/aws/script.sh
```

#### 9. Configure Logrotate as per Requirement
Edit the logrotate configuration for NGINX:

```sh
vi /etc/logrotate.d/nginx
```

Add the following configuration:

```sh
/var/log/nginx/*.log {
    create 0640 nginx root
    daily
    rotate 10
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /run/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

#### 10. Script for Logrotate
Create a script for log rotation:

```sh
cd /root/aws 
vi logrotate.sh
```

Add the following content to `logrotate.sh`:

```bash
#!/bin/bash

logrotate -f /etc/logrotate.conf
```

Make the script executable:

```sh
chmod -R 777 logrotate.sh
```

#### 11. Setup Crontab for Logrotate
Edit the crontab to schedule the logrotate script:

```sh
crontab -e
```

Add the following line to run the script daily at 11 AM:

```sh
0 11 * * * sh /root/aws/logrotate.sh
```

Verify the crontab entry:

```sh
crontab -l
```

### Additional Notes
- Ensure that the security group associated with your EC2 instance allows traffic on ports 80 (HTTP) and 8080.
- Make sure Docker and Docker-Compose are correctly installed and running.

---

This guide helps you deploy WordPress with Docker and Docker-Compose on an EC2 instance and use NGINX as a reverse proxy to manage the site.
