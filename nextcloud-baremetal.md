# Installation of nextcloud as bare metal directly on the OS

## Install apache and php7.2

I plan to run on apache2, basic ubuntu level

```
aptitude -y install apache2
systemctl start apache2
aptitude -y install libapache2-mod-php7.2 php7.2-gd php7.2-json php7.2-mysql \
php7.2-curl php7.2-mbstring php7.2-intl php-imagick php7.2-xml php7.2-zip \
libxmlrpc-epi0 php7.2 php7.2-gmp php7.2-xmlrpc
```
Configure apache for nextcloud
```
sed -i "s/Options Indexes FollowSymLinks/Options FollowSymLinks/" /etc/apache2/apache2.conf
```
Configure php
```
sed -i "s/memory_limit = 128M/memory_limit = 512M/" /etc/php/7.2/apache2/php.ini
sed -i "s/upload_max_filesize = 2M/upload_max_filesize = 200M/" /etc/php/7.2/apache2/php.ini
sed -i "s/max_execution_time = 30/max_execution_time = 360/" /etc/php/7.2/apache2/php.ini
sed -i "s/post_max_size = 8M/post_max_size = 200M/" /etc/php/7.2/apache2/php.ini 
sed -i "s/date.timezone = /date.timezone = Europe\/Berlin/" /etc/php/7.2/apache2/php.ini
sed -i "s/memory_limit = 128M/memory_limit = 512M/" /etc/php/7.2/cli/php.ini
sed -i "s/upload_max_filesize = 2M/upload_max_filesize = 200M/" /etc/php/7.2/cli/php.ini
sed -i "s/max_execution_time = 30/max_execution_time = 360/" /etc/php/7.2/cli/php.ini
sed -i "s/post_max_size = 8M/post_max_size = 200M/" /etc/php/7.2/cli/php.ini 
sed -i "s/date.timezone = /date.timezone = Europe\/Berlin/" /etc/php/7.2/cli/php.ini
```
Activate apache mods
```
a2enmod rewrite
a2enmod headers
a2enmod env
a2enmod dir
a2enmod mime
a2enmod ssl
systemctl restart apache2

```

## Install mariadb and prepare database for nextcloud

Install ubuntu packaged
```
aptitude -y install mariadb-server mariadb-client
```
Configure mariadb. The utf8mb4 and barracuda settings are important for the database format later on.
```
systemctl stop mariadb
sed -i '/\[mysqld\]/a transaction_isolation = READ-COMMITTED\nbinlog_format = ROW' /etc/mysql/mariadb.conf.d/50-server.cnf
sed -i '/\[mysqld\]/a character-set-server = utf8mb4\ncollation-server = utf8mb4_general_ci' /etc/mysql/mariadb.conf.d/50-server.cnf
sed -i '/\[mysqld\]/a innodb_large_prefix=on\ninnodb_file_format=barracuda\ninnodb_file_per_table=1' /etc/mysql/mariadb.conf.d/50-server.cnf
systemctl start mariadb
```
Secure the installation. Say yes to everything
```
mysql_secure_installation
```
Now create the nextcloud db and user
```
mysql -u root --password=<goodpassword>
CREATE USER 'nextcloud'@'localhost' IDENTIFIED BY '<goodpassword>';
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES on nextcloud.* to 'nextcloud'@'localhost';
FLUSH privileges;
EXIT;
```

## Configure nextcloud setup

Create phpinfo file
```
sudo cat <<EOF >> /var/www/html/phpinfo.php
<?php phpinfo( ); ?>
EOF
```
Configure the virtual host for nextcloud
```
sudo cat <<EOF >> /etc/apache2/sites-available/nextcloud.conf
<VirtualHost *:80> 
    ServerAdmin <myemail> 
    DocumentRoot /var/www/html/nextcloud/ 
    ServerName <myserver> 

    Alias /nextcloud "/var/www/html/nextcloud/" 

    <Directory /var/www/html/nextcloud/> 
        Options Indexes Multiviews FollowSymlinks 
        AllowOverride All 
        Order allow,deny 
        Require all granted
        Allow from all
 
        <IfModule mod_dav.c> 
            Dav off 
        </IfModule> 
        <IfModule mod_headers.c>
            Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
        </IfModule>

       SetEnv HOME /var/www/html/nextcloud/ 
       SetEnv HTTP_HOME /var/www/html/nextcloud/ 
    </Directory> 

    ErrorLog \${APACHE_LOG_DIR}/error.log 
    CustomLog \${APACHE_LOG_DIR}/access.log combined 

</VirtualHost>
EOF
```
Disable the defaul site and enable the nextcloud site
```
a2dissite 000-default
a2ensite nextcloud
systemctl reload apache2
```

Prepare the directory, get nextcloud and extract
```
mkdir -p /var/www/html
cd /var/www/html
wget https://download.nextcloud.com/server/releases/nextcloud-18.0.2.zip
unzip nextcloud-18.0.2.zip
sudo chown -R www-data:www-data /var/www/html/nextcloud
```

Small comment, if you delete the database to recreate, also delete www dir and extract new

Now let's install nextcloud
```
sudo -u www-data php occ  maintenance:install --database \
"mysql" --database-name "nextcloud"  --database-user "nextcloud" --database-pass \
"<goodpassword>" --admin-user "admin" --admin-pass "<goodpassword>"
```

## letscertify

Install certbot
```
aptitude -y install software-properties-common
add-apt-repository ppa:certbot/certbot
aptitude -y install certbot python3-certbot-apache
```
Request certs, NAT router has to be set up
```
certbot --apache -m <myemail> -d <myserver> --agree-tos
```
weekly renewal of certs. Edit crontab `crontab -e`and add 
```
MAILTO=<myemail>
0 2 * * Sun root /usr/bin/certbot renew
```

## Tuning nextcloud

Add .htaccess to apache
```
sudo -u www-data cat <<EOF >> /mirror/www/html/.htaccess
<IfModule mod_rewrite.c>
  RewriteEngine on
  RewriteRule ^\.well-known/host-meta /var/www/html/nextcloud/public.php?service=host-meta [QSA,L]
  RewriteRule ^\.well-known/host-meta\.json /var/www/html/nextcloud/public.php?service=host-meta-json [QSA,L]
  RewriteRule ^\.well-known/webfinger /var/www/html/nextcloud/public.php?service=webfinger [QSA,L]
  RewriteRule ^\.well-known/carddav /var/www/html/nextcloud/remote.php/dav/ [R=301,L]
  RewriteRule ^\.well-known/caldav /var/www/html/nextcloud/remote.php/dav/ [R=301,L]
</IfModule>
EOF
systemctl start apache2
```
Setup nextcloud trusted domains and fix db setup
```
sudo -u www-data php occ config:system:get trusted_domains
sudo -u www-data php occ config:system:set trusted_domains 2 --value=<myserver>
sudo -u www-data php occ db:add-missing-indices
sudo -u www-data phpocc db:convert-filecache-bigint
```
Add nextcloud cron entry `*/5 * * * * php -f /var/www/nextcloud/cron.php > /dev/null 2>&1
` to crontab
```
sudo crontab -u www-data -e
```
Install redis and php cache and configure it
```
sudo systemctl stop apache2
sudo aptitude -y install redis php7.2-redis php7.2-apcu
sudo sed -i "/instanceid/a\ \ 'memcache.local' => '\OC\Memcache\APCu'," /var/www/html/nextcloud/config/config.php
sudo sed -i "/memcache.local/a\ \ 'memcache.distributed' => '\OC\Memcache\Redis'," /var/www/html/nextcloud/config/config.php
sudo sed -i "/memcache.distributed/a\ \ 'redis' => \[" /var/www/html/nextcloud/config/config.php
sudo sed -i "/redis/a\ \ \ \ 'host'     => '/var/run/redis/redis.sock'," /var/www/html/nextcloud/config/config.php
sudo sed -i "/redis.sock/a\ \ \ \ 'port'     => 0," /var/www/html/nextcloud/config/config.php
```
Change the log time format so that fail2ban can read it and set the log path
```
sudo sed -i "/  'logdateformat' => */  'logdateformat' => 'Y-m-d H:i:s'," /var/www/html/nextcloud/config/config.php
sudo sed -i "/  logfile' => */  'logfile' => '/var/log/nextcloud/nextcloud.log'," /var/www/html/nextcloud/config/config.php
sudo mkdir /var/log/nextcloud
sudo chown -R www-data:www-data /var/log/nextcloud
```

Setup logrotate for nextcloud log
```
sudo cat <<EOF > /etc/logrotate.d/nextcloud
/var/log/nextcloud/*.log {
rotate 12
monthly
compress
missingok
notifempty
}
EOF
```

Set up redis config and socket
```
usermod -a -G redis www-data
sed -i "s/#unixsocket \/var\/run\/redis\/redis-server.sock/unixsocket \/var\/run\/redis\/redis.sock" /etc/apache2/apache2.conf
sed -i "s/#unixsocketperm 770/unixsocketperm 770/" /etc/apache2/apache2.conf
systemctl restart redis
systemctl start apache2
```
Enable OPcache in php
```
systemctl stop apache2
sed -i "s/opcache.enable=.*/opcache.enable=1/" /etc/php/7.2/apache2/php.ini
sed -i "s/opcache.interned_strings_buffer=*./opcache.interned_strings_buffer=8/" /etc/php/7.2/apache2/php.ini
sed -i "s/opcache.max_accelerated_files=.*/opcache.max_accelerated_files=10000/" /etc/php/7.2/apache2/php.ini
sed -i "s/opcache.memory_consumption=.*/opcache.memory_consumption=128/" /etc/php/7.2/apache2/php.ini
sed -i "s/opcache.save_comments=.*/opcache.save_comments=1/" /etc/php/7.2/apache2/php.ini
sed -i "s/opcache.revalidate_freq=.*/opcache.revalidate_freq=1/" /etc/php/7.2/apache2/php.ini
systemctl start apache2
```

## Setting up email notification
Install tools
```
sudo aptitude -y install msmtp msmtp-mta mailutils
```
Configure mail sending
```
sudo cat <<EOF > /etc/msmtprc
defaults
port 587
tls on
tls_starttls on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
account <myemail>
host <mysmtp>
from <myemail>
auth on
user <mysmtpuser>
password <mystpmpwd>
account default: <myemail>
aliases /etc/aliases
logfile /var/log/msmtp/msmtp.log
EOF
```
Secure access to config file with secrets and log
```
sudo chmod 600 /etc/msmtprc
sudo mkdir /var/log/msmtp
sudo chown -R www-data:adm /var/log/msmtp
```
Setup logrotate
```
sudo cat <<EOF > /etc/logrotate.d/msmtp
/var/log/msmtp/*.log {
rotate 12
monthly
compress
missingok
notifempty
}
EOF
```
Setup general alias
```
sudo cat <<EOF > /etc/aliases
root: <myemail>
default: <myemail>
EOF
sudo cat <<EOF > /etc/mail.rc
set sendmail="/usr/bin/msmtp -t"
EOF
```

## Firewall setup for nextcloud

Allow generic access to nextcloud
```
ufw allow 80/tcp
ufw allow 443/tcp
```

## fail2ban setup for nextcloud

Setup filter. Last filter is against Trusted domain errors.
```
cat <<EOF >/etc/fail2ban/filter.d/nextcloud.conf
[Definition]
failregex=^{"reqId":".*","remoteAddr":".*","app":"core","message":"Login failed: '.*' \(Remote IP: '<HOST>'\)","level":2,"time":".*"}$
          ^{"reqId":".*","level":2,"time":".*","remoteAddr":".*","user,:".*","app":"no app in context".*","method":".*","message":"Login failed: '.*' \(Remote IP: '<HOST>'\)".*}$
          ^{"reqId":".*","level":2,"time":".*","remoteAddr":".*","user":".*","app":".*","method":".*","url":".*","message":"Login failed: .* \(Remote IP: <HOST>\).*}$
          ^{"reqId":".*","level":1,"time":".*","remoteAddr":"<HOST>","user":".*","app":".*","method":".*","url":".*","message":"Trusted domain error.*}$
EOF
```
Setup jail
```
cat <<EOF >/etc/fail2ban/jail.d/nextcloud.local
[nextcloud]
backend = auto
enabled = true
port = 80,443
protocol = tcp
filter = nextcloud
maxretry = 10
bantime = 36000
findtime = 3600
logpath = /var/www/html/nextcloud/nextcloud.log
EOF
```

Setup email notification for fail2ban
```
sed -i "/DEFAULT/a ignorself = true" /etc/fail2ban/jail.local
sed -i "/ignorself = true/a destemail = <myemail>" /etc/fail2ban/jail.local
sed -i "/destemail = .*/a sender = <myemail>" /etc/fail2ban/jail.local
sed -i "/sender = .*/a mta = mail" /etc/fail2ban/jail.local
sed -i "/mta = mail/a action = %(action_mw)s" /etc/fail2ban/jail.local
```

Setup actions for email and copy to all actions
```
sudo cat <<EOF > /etc/fail2ban/action.d/mail-buffered.local
[Definition]
actionstart =
actionstop =
EOF
cp /etc/fail2ban/action.d/mail-buffered.local /etc/fail2ban/action.d/mail.local
cp /etc/fail2ban/action.d/mail-buffered.local /etc/fail2ban/action.d/mail-whois-lines.local
cp /etc/fail2ban/action.d/mail-buffered.local /etc/fail2ban/action.d/mail-whois.local
cp /etc/fail2ban/action.d/mail-buffered.local /etc/fail2ban/action.d/sendmail-buffered.local
cp /etc/fail2ban/action.d/mail-buffered.local /etc/fail2ban/action.d/sendmail-common.local
systemctl restart fail2ban
```
## scripts for automatic upgrade and backup

### nextcloud upgrade

Add script for upgrade
```
sudo cat<<EOF > /usr/local/bin/nextcloud_backup.sh
#!/bin/bash
echo "Upgrade started, $(date)"
return=$(sudo -u www-data php /var/www/html/nextcloud/occ update:check)
if [[ "$return" == "Everything up to date" ]]
then
    echo "no update necessary"
    exit 0
fi

sudo -u www-data php /var/www/html/nextcloud/updater/updater.phar --no-interaction
sudo -u www-data php /var/www/html/nextcloud/occ status
sudo -u www-data php /var/www/html/nextcloud/occ -V
sudo -u www-data php /var/www/html/nextcloud/occ db:add-missing-indices
sudo -u www-data php /var/www/html/nextcloud/occ db:convert-filecache-bigint
sed -i "s/output_buffering=.*/output_buffering='Off'/" /var/www/html/nextcloud/.user.ini
chown -R www-data:www-data /var/www/html/nextcloud
redis-cli -s /var/run/redis/redis.sock <<EOL
FLUSHALL
quit
EOL
sudo -u www-data php /var/www/html/nextcloud/occ files:scan --all
sudo -u www-data php /var/www/html/nextcloud/occ files:scan-app-data
sudo -u www-data php /var/www/html/nextcloud/occ update:check
sudo -u www-data php /var/www/html/nextcloud/occ app:update --all
systemctl restart apache2
echo "Upgrade finished, $(date)"
EOF
chmod 755 /usr/local/bin/nextcloud_upgrade.sh
```

### nextcloud automatic backup

Add script for backup
```
sudo cat<<EOF > /usr/local/bin/nextcloud_backup.sh
source /root/.profile
echo "Backup starting $(date)"
export DATUM=$(date +"%F")
echo "Directory: $DATUM"
path="/mirror/backups/$DATUM"
if [ ! -d "$path" ]
then
    /bin/mkdir -p $path
fi
/bin/systemctl stop apache2
/usr/bin/sudo -u www-data php /var/www/html/nextcloud/occ maintenance:mode --on
/bin/tar -cpzf $path/ncserver.tar.gz -C /var/www/html/nextcloud .
/usr/bin/mysqldump --single-transaction -h localhost -u nextcloud -p<goodpassword> nextcloud > $path/ncdb.sql
/usr/bin/sudo -u www-data php /var/www/html/nextcloud/occ maintenance:mode --off
/usr/bin/redis-cli -s /var/run/redis/redis.sock <<EOL
FLUSHALL
quit
EOL
/usr/bin/sudo -u www-data php /var/www/html/nextcloud/occ files:scan --all
/usr/bin/sudo -u www-data php /var/www/html/nextcloud/occ files:scan-app-data
/usr/bin/sudo -u www-data php /var/www/html/nextcloud/occ app:update --all
/bin/systemctl start apache2
echo "Backup finished $(date)"
EOF
chmod 755 /usr/local/bin/nextcloud_backup.sh
```

### Adding scripts to cron

Excute `crontab -e`
and add this
```
MAILTO=<myemail>
30 2 * * Sun /usr/local/bin/nextcloud_backup.sh 2>&1 >/var/log/nextcloud/backup.log
MAILTO=<myemail>
0 3 * * Sun /usr/local/bin/nextcloud_upgrade.sh 2>&1 >/var/log/nextcloud/upgrade.log
```

### Script to restore

This script should restore the nextcloud from a backup. It needs the directory of the backup as parameter
```
sudo cat<<EOF > /usr/local/bin/nextcloud_restore.sh
DATUM="$1"
source /root/.profile

sudo -u www-data php /var/www/html/nextcloud/occ maintenance:mode --on
systemctl stop apache2
rm -r /var/www/html/nextcloud/
mkdir -p /var/www/html/nextcloud/
tar -xpzf /mirror/backups/$DATUM/ncserver.tar.gz -C /var/www/html/nextcloud/
chown -R www-data:www-data /var/www
mysql -h localhost -uroot -p<goodpassword> -e "DROP DATABASE nextcloud"
mysql -h localhost -uroot -p<goodpassword> -e "CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci"
mysql -h localhost -uroot -p<goodpassword> -e "GRANT ALL PRIVILEGES on nextcloud.* to nextcloud@localhost"
mysql -h localhost -unextcloud -p<goodpassword> nextcloud < /mirror/backups/$DATUM/ncdb.sql
sudo -u www-data php /var/www/html/nextcloud/occ maintenance:mode --off
/usr/bin/redis-cli -s /var/run/redis/redis.sock <<EOL
FLUSHALL
quit
EOL
/usr/bin/sudo -u www-data php /var/www/html/nextcloud/occ files:scan --all
/usr/bin/sudo -u www-data php /var/www/html/nextcloud/occ files:scan-app-data
/usr/bin/sudo -u www-data php /var/www/html/nextcloud/occ app:update --all
systemctl restart apache2
EOF
chmod 755 /usr/local/bin/nextcloud_restore.sh
```

## nextcloud configs

At the end, login as administrator to your nextcloud and click on you user in the top right. In the drop down, click on Settings. In Overview, nextcloud will check if your installation is healthy. It should be :-)

Under basic settings, add your email settings so that your nextcloud can send emails.

Then go to personal settings and adapt for your admin language, email address and profile picture.