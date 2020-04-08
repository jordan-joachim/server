# failed - Nextcloud on docker-compose

First, i tried to run the basic example using jwilders nginx proxy and letsencrypt image... But letsencrypt failed, could not verify the certificate.

Then i dug a little bit deeper and actually wanted to have a setup that did not expose the docker socket to any container to avoid security issues.

For that, i decided to run the nginx reverse proxy directly on the OS

## nginx install and setup

Installation
```
sudo apt-get -y install nginx
```

Configuration of my site
```
sudo cat <<EOF >>  /etc/nginx/sites-available/<myserver>.conf
server {
    server_name <myserver>;
    root /var/www/<myserver>;

    index index.html;

    location / {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
        client_max_body_size 0;

        access_log /var/log/nginx/nextcloud.access.log;
        error_log /var/log/nginx/nextcloud.error.log;
    }

    location /.well-known/carddav {
      return 301 $scheme://$host/remote.php/dav;
    }

    location /.well-known/caldav {
      return 301 $scheme://$host/remote.php/dav;
    }

    listen [::]:443 ssl http2; # managed by Certbot
    listen 443 ssl http2; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/<myserver>/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/<myserver>/privkey.pem; # managed by Certbot
    ssl_trusted_certificate /etc/letsencrypt/live/<myserver>/chain.pem;
    include /etc/nginx/snippets/ssl.conf;

}

server {
    if ($host = <myserver>) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

        listen 80;
        listen [::]:80;

        server_name <myserver>;
    return 404; # managed by Certbot
}
EOF
```
Additional files with security configurations
```
 sudo cat <<EOF >> /etc/nginx/snippets/ssl.conf 
# Hide the nginx version
server_tokens off;
# Disable gzip due the HTTPS BREACH attack. Only activate if it is really required by your application.
gzip off;

# SSL Settings
# The generated DH Group
ssl_dhparam /etc/ssl/certs/dhparam.pem;

ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ecdh_curve secp384r1;
ssl_prefer_server_ciphers on;

# Ciphersuites recommendation from the chiper.li
# Use this chipersuites to get 100 points of the SSLabs test
# Some device will not support
# ssl_ciphers "ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384";

# Mozilla Ciphersuits Recommendation
# Use this for all devices supports
ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';

ssl_session_timeout 10m;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;

# OSCP stapling
ssl_stapling on;
ssl_stapling_verify on;
EOF
```

```
  sudo cat <<EOF >> /etc/nginx/snippets/header.con 
HSTS Header, only use HTTPS, applied on all subdomains
add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;";
add_header X-Robots-Tag none;
# Disable mime content-type sniffing
add_header X-Download-Options noopen;
add_header X-Permitted-Cross-Domain-Policies none;
add_header X-Content-Type-Options "nosniff" always;
# Deny your page to be embedded in frames / iframes to protect from clickjacking
add_header X-Frame-Options DENY;
# Enable Cross-Site scripting protection
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "no-referrer" always;
add_header Feature-Policy "geolocation 'self'";
EOF
```

Create pem file for nginx
```
openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096
```

## certbot setup for nginx

Install certbot repo and install certbot for ngins

```
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get install certbot nginx python3-certbot-nginx -y
```
Create certificates
```
sudo certbot --nginx -m <mailid> -d <myserver> --agree-tos
```
Add renew to crontab
```
crontab -e
```
and add this `0 2 * * Sun root /usr/bin/certbot renew` 

Restart nginx
```
sudo systemctl restart nginx
```

## Install docker and docker-compose

Install prereqs
```
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
```
Add docker repos
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository  "deb [arch=amd64] https://download.docker.com/linux/ubuntu  $(lsb_release -cs) stable"
sudo apt-get update
```
Install docker
```
sudo apt-get -y install docker-ce
```
Allow your user access to docker
```
sudo usermod -aG docker <youruser>
```
Download and install docker-compose
```
sudo curl -s -L https://github.com/docker/compose/releases/download/1.25.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## prepare nextcloud docker-compose env

Clone nextcloud docker repository
```
sudo git clone https://github.com/nextcloud/docker.git
```

Create directories for local files
```
sudo mkdir -p /mirror/docker/html
sudo mkdir -p /mirror/docker/mysql  
sudo mkdir -p /mirror/docker/nextcloud
```

Prepare db.env file
```
sudo cat <<EOF >> /mirror/docker/nextcloud/db.env
MYSQL_PASSWORD=<agoodpassword>
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
EOF
```
write out the docker-compose yaml
```
sudo cat <<EOF >> /mirror/docker/nextcloud/docker-compose.yml
version: '3'

services:

  mariadb:
    image: mariadb
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    restart: always
    volumes:
      - /mirror/docker/mysql/:/var/lib/mysql/
    environment:
      - MYSQL_ROOT_PASSWORD=<agoodpassword>
    env_file:
      - db.env
    expose:
      - "3306"

  redis:
    image: redis:alpine
    restart: always
    expose: 
      - "6379"

  apache:
    image: nextcloud:apache 
    restart: always
    volumes:
      - /mirror/docker/html/:/var/www/html/
    environment:
      - MYSQL_HOST=mariadb
      - REDIS_HOST=redis
    env_file:
      - db.env
    depends_on:
      - mariadb
      - redis
    ports:
      - 8080:80

  cron:
    image: nextcloud:apache
    restart: always
    volumes:
      - /mirror/docker/html/:/var/www/html/
    entrypoint: /cron.sh
    depends_on:
      - mariadb
      - redis
EOF
```

## Starting nextcloud docker-compose

I started the docker images with
```
cd /mirror/docker/nextcloud
docker-compose up -d
```

After that, i can access the nextcloud installation fine via the internal exposed port 8080. I found that there were some warnings, so to fix this i ran
```
docker-compose exec --user www-data apache ./occ db:add-missing-indices
docker-compose exec --user www-data apache ./cc db:convert-filecache-bigint
```

But external access via <myserver> did not work completely. Also file share clients seemed to have problems.

After that i stopped trying to run this in docker-compose and installed nextcloud natively.

If i try to run nextcloud again containerized, i'll try a single node kubernetes.

