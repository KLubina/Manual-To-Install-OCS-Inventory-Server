# OCS Inventory Server Setup Guide (Debian 12)

## Prerequisites

- Debian 12 Server
- Root access
- Internet connection

## Step 1: Root Access

su -

## Step 2: MariaDB Database Setup

### Install and start MariaDB

apt install mariadb-server mariadb-common mariadb-client
systemctl enable mariadb
systemctl start mariadb

### Create database and user

mysql -u root

Execute in MySQL console:

CREATE DATABASE ocsweb;
CREATE USER 'ocs'@'localhost' IDENTIFIED BY 'ocs';
GRANT ALL PRIVILEGES ON ocsweb.* TO 'ocs'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit;

## Step 3: Install OCS Inventory Server

### Add repository and install

apt install -y curl
curl -sS https://deb.ocsinventory-ng.org/pubkey.gpg | sudo apt-key add -
echo "deb http://deb.ocsinventory-ng.org/debian/ bookworm main" | sudo tee /etc/apt/sources.list.d/ocsinventory.list
apt update
apt install ocsinventory

### Set permissions

chmod -R 777 /usr/share/ocsinventory-reports/ /var/lib/ocsinventory-reports/ /etc/ocsinventory-server/ /var/log/ocsinventory-server/

### Remove install.php

rm /usr/share/ocsinventory-reports/ocsreports/install.php

## Step 4: Adjust PHP Limits

### Edit PHP configuration

nano /etc/php/*/apache2/php.ini

**Find and adjust the following values:**

- **post_max_size** (line ~946): Set to desired size (e.g. `0` which stands for unlimited)
- **upload_max_filesize** (line ~925): Set to desired size
- **memory_limit** (line ~491): Set to desired size

### Restart Apache

systemctl restart apache2

## Step 5: Enable SSL

### Install OpenSSL

apt install openssl -y

### Create SSL certificate

openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr

**During certificate creation:**
- **Common Name**: `192.83.247.19` (enter your server IP!)

### Create self-signed certificate

openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

### Copy certificates

cp server.crt /etc/ssl/certs/
cp server.key /etc/ssl/private/

### Enable SSL modules

a2enmod ssl
a2ensite default-ssl

### Adjust SSL configuration

nano /etc/apache2/sites-enabled/default-ssl.conf

**Find and change these lines:**

SSLCertificateFile /etc/ssl/certs/server.crt
SSLCertificateKeyFile /etc/ssl/private/server.key


**Add BEFORE `</VirtualHost>`:**

Alias /download /var/lib/ocsinventory-reports/download
<Directory /var/lib/ocsinventory-reports/download>
    <IfModule mod_authz_core.c>
        Require all granted
    </IfModule>
    <IfModule !mod_authz_core.c>
        Order allow,deny
        Allow from all
    </IfModule>
</Directory>

## Step 6: Enable Software Deployment

### Setup download directory

mkdir -p /var/lib/ocsinventory-reports/download
chown www-data:www-data /var/lib/ocsinventory-reports/download
chmod 755 /var/lib/ocsinventory-reports/download

### Provide certificate for agent download

cp /etc/ssl/certs/server.crt /var/www/html/cacert.pem
chmod 644 /var/www/html/cacert.pem

### Restart Apache

service apache2 restart

## Further Infos

### Access OCS Inventory

**Web Interface**: `https://192.83.247.19/ocsreports`

**Default Login:**
- **Username**: `admin`
- **Password**: `admin`

### Download Certificate File

The certificate file is available for download at:
http://192.83.247.19/cacert.pem

### Settings for Software Deployment

Change in general configuration:

- **DOWNLOAD**: `ON`
- **DOWNLOAD_URI_FRAG**: `http://192.83.247.19/download`
- **DOWNLOAD_URI_INFO**: `https://192.83.247.19/download`

1. **Deployment → Build**
2. **Deployment → Activate → Active**
3. **Assign package**: Select computer → Deploy → Assign package

### Logging and Troubleshooting

#### Apache logs
tail -f /var/log/apache2/access.log
tail -f /var/log/apache2/error.log

#### OCS server logs
tail -f /var/log/ocsinventory-server/activity.log

#### OCS Agent logs
type "C:\ProgramData\OCS Inventory NG\Agent\ocsinventory.log"

#### Download logs
type "C:\ProgramData\OCS Inventory NG\Agent\download.log"
