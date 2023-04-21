## GLPI Docker Image 🐋
Image is based on [Alpine 3.17](https://hub.docker.com/repository/docker/johann8/alpine-glpi/general)

| pull | size | version | platform |
|:---------------------------------:|:----------------------------------:|:--------------------------------:|:--------------------------------:|
| ![Docker Pulls](https://img.shields.io/docker/pulls/johann8/alpine-glpi?style=flat-square) | ![Docker Image Size (latest by date)](https://img.shields.io/docker/image-size/johann8/alpine-glpi/latest) | [![](https://img.shields.io/docker/v/johann8/alpine-glpi?sort=date)](https://hub.docker.com/r/johann8/alpine-glpi/tags "Version badge") | ![](https://img.shields.io/badge/platform-amd64-blue "Platform badge") |

## OCS Inventoryi NG Docker Image 🐋
Image is based on [Alpine 3.17](https://hub.docker.com/repository/docker/johann8/alpine-ocs/general)

| pull | size | version | platform |
|:---------------------------------:|:----------------------------------:|:--------------------------------:|:--------------------------------:|
| ![Docker Pulls](https://img.shields.io/docker/pulls/johann8/alpine-ocs?style=flat-square) | ![Docker Image Size (latest by date)](https://img.shields.io/docker/image-size/johann8/alpine-ocs/latest) | [![](https://img.shields.io/docker/v/johann8/alpine-ocs?sort=date)](https://hub.docker.com/r/johann8/alpine-ocs/tags "Version badge") | ![](https://img.shields.io/badge/platform-amd64-blue "Platform badge") |


<h1 align="center">GLPI - IT Asset Management</h1>
<p align='justify'>

<a href="https://glpi-project.org">GLPI</a> - is an open source IT Asset Management, issue tracking system and service desk system. This software is written in PHP and distributed as open-source software under the GNU General Public License.

GLPI is a web-based application helping companies to manage their information system. The solution is able to build an inventory of all the organization's assets and to manage administrative and financial tasks. The system's functionalities help IT Administrators to create a database of technical resources, as well as a management and history of maintenances actions. Users can declare incidents or requests (based on asset or not) thanks to the Helpdesk feature.
</p>

<h1 align="center">OCS Inventory NG</h1>
<p align='justify'>
<a href="https://glpi-project.org">OCS Inventory NG</a> - (Open Computers and Software Inventory Next Generation) is an assets management and deployment solution.
Since 2001, OCS Inventory NG has been looking for making software and hardware more powerful.
OCS Inventory NG asks its agents to know the software and hardware composition of every computer or server.
</p>


<h1 align="center">Installation and Configuration</h1>

- [Install GLPI and OCS Inventory docker container](#install-glpi-and-ocs-Inventory-docker-container)
- [Install OCS Inventory](#install-ocs-inventory)
  - [OCS Inventory configuration](#ocs-inventory-setup)
- [Install GLPI](#install-glpi)
  - [GLPI configuration](#glpi-setup)
- [Database backup](#database-backup-)
- [Database restore](#database-restore-)
- [SMTP setup on docker host](#smtp-setup-on-docker-host)

## Install GLPI and OCS Inventory docker container
- create folders

```bash
DOCKERDIR=/opt/inventory
mkdir -p ${DOCKERDIR}/data/ocsinventory/{perlcomdata,ocsreportsdata,varlibdata,httpdconfdata} 
mkdir -p ${DOCKERDIR}/data/nginx/{config,certs,auth}
chown -R 101:101 ${DOCKERDIR}/data/ocsinventory/perlcomdata/
chown -R 101:101 ${DOCKERDIR}/data/ocsinventory/ocsreportsdata/
chown -R 101:101 ${DOCKERDIR}/data/ocsinventory/varlibdata/
mkdir -p ${DOCKERDIR}/data/{glpi,crond,crontabs,mariadb}
mkdir -p ${DOCKERDIR}/data/glpi/{files,plugins,config}
mkdir -p ${DOCKERDIR}/data/crond/{5min,15min,30min,hourly,daily,weekly,monthly}
chown -R 100:101 ${DOCKERDIR}/data/glpi/*
mkdir -p ${DOCKERDIR}/data/mariadb/{config,dbdata,socket}
cd ${DOCKERDIR}
tree -d -L 3 ${DOCKERDIR}
```

- Download config files

```bash
DOCKERDIR=/opt/inventory
cd ${DOCKERDIR}
wget https://raw.githubusercontent.com/johann8/Inventory-OCS-GLPI/master/docker-compose.yml
wget https://raw.githubusercontent.com/johann8/Inventory-OCS-GLPI/master/docker-compose.override.yml
wget https://raw.githubusercontent.com/johann8/Inventory-OCS-GLPI/master/.env
wget https://raw.githubusercontent.com/johann8/Inventory-OCS-GLPI/master/assets/mariadb/config/my.cnf -P data/mariadb/config/
wget https://raw.githubusercontent.com/johann8/Inventory-OCS-GLPI/master/assets/nginx/config/ocsinventory.conf.template -P data/nginx/config
```

- Customize variable in .env file

- Generate passwords for MariaDB
```bash
DOCKERDIR=/opt/inventory
PASSWORD_OCS=$(pwgen -1cnsB 25 1); sed -i "s/MyPassword123/${PASSWORD_OCS}/" ${DOCKERDIR}/.env
PASSWORD_GLPI=$(pwgen -1cnsB 25 1); sed -i "s/MyPassword456/${PASSWORD_GLPI}/" ${DOCKERDIR}/.env 
PASSWORD_ROOT=$(pwgen -1cnsB 30 1); sed -i "s/MySuperPassword789/${PASSWORD_ROOT}/" ${DOCKERDIR}/.env
cat ${DOCKERDIR}/.env
```
- Generate Nginx certificate for FQDN: ocsinventory.changeme.de
```bash
# Generate private key
openssl genrsa -out /etc/pki/tls/private/ca.key 2048 

# Generate CSR (Common Name is ocsinventory.changeme.de)
openssl req -new -key /etc/pki/tls/private/ca.key -out /etc/pki/tls/private/ca.csr

# Generate Self Signed Key
openssl x509 -req -days 3650 -in /etc/pki/tls/private/ca.csr -signkey /etc/pki/tls/private/ca.key -out /etc/pki/tls/certs/ca.crt
openssl x509 -in  /etc/pki/tls/certs/ca.crt -text -noout

# convert crt to pem
cd /etc/pki/tls/certs && openssl x509 -in ca.crt -out cacert.pem
cd -
openssl x509 -in  /etc/pki/tls/certs/cacert.pem -text -noout

# copy certificates
DOCKERDIR=/opt/inventory
cp /etc/pki/tls/private/ca.key ${DOCKERDIR}/data/nginx/certs/ocs.key
cp /etc/pki/tls/certs/ca.crt ${DOCKERDIR}/data/nginx/certs/ocs.crt
cp /etc/pki/tls/certs/cacert.pem ${DOCKERDIR}/
```
- Run all docker container

```bash
DOCKERDIR=/opt/inventory
cd ${DOCKERDIR}
docker-compose up -d

# show logs
docker-compose logs

# show running containers
docker-compose ps
```
- Create GLPI Database
```bash
DOCKERDIR=/opt/inventory
docker-compose exec mariadb bash
mysql --batch --user=root --password=${MARIADB_ROOT_PASSWORD} -e "create database "${MARIADB_GLPI_DATABASE}" character set utf8mb4"
mysql --batch --user=root --password=${MARIADB_ROOT_PASSWORD} -e "CREATE USER "${MARIADB_GLPI_USER}""
mysql --batch --user=root --password=${MARIADB_ROOT_PASSWORD} -e "grant all on "${MARIADB_GLPI_DATABASE}".*  to '${MARIADB_GLPI_USER}'@'%' identified by '${MARIADB_GLPI_PASSWORD}'"
mysql --batch --user=root --password=${MARIADB_ROOT_PASSWORD} -e "FLUSH PRIVILEGES"
mysql --batch --user=root --password=${MARIADB_ROOT_PASSWORD} -e "show databases;"
mysql --batch --user=root --password=${MARIADB_ROOT_PASSWORD} -e "select Host,User,Password from mysql.user;"
exit
```
### Install OCS Inventory
- Go to http://ocs.changeme.de/ocsreports/
- Run through the installation wizard and log in with `admin / admin`

### Install GLPI
- Go to http://glpi.changme.de
- Enter the database connection details as shown in the picture
![Connect to Database](https://raw.githubusercontent.com/johann8/Inventory-OCS-GLPI/master/docs/assets/screenshots/GLPI_Database_Setup_1.PNG)
- Choose the database glpi
![Choose Database](https://raw.githubusercontent.com/johann8/Inventory-OCS-GLPI/master/docs/assets/screenshots/GLPI_Database_Setup_2.PNG)

- Run through the installation wizard and log in with `glpi / glpi`

### OCS Inventory setup
- Setup of `OCS Inventory` can be found [here](https://github.com/johann8/ocs-alpine)

### GLPI setup
- Setup of `GLPI` can be found [here](https://github.com/johann8/alpine-glpi)

</br>

---

> :warning: Please do not forget to change `docker-compose.yml` after installation as follows
```bash
# change docker-compose.yml - will delete install.php
DOCKERDIR=/opt/inventory
cd ${DOCKERDIR}
vim docker-compose.yml
----------------------
change from
...
OCS_INVENTOTRY_INSTALL: true
...

to
...
OCS_INVENTOTRY_INSTALL: false
...
----------------------
```
## Database backup ![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)

You can backup the database with this <a href="https://github.com/johann8/tools/tree/master/mariadb">script</a>.

- Download bash script
```bash
wget https://raw.githubusercontent.com/johann8/tools/master/mariadb/mysqldump_docker_backup_schema.sh -P /usr/local/bin/
wget https://raw.githubusercontent.com/johann8/tools/master/mariadb/mysqldump_docker_backup_full.sh -P /usr/local/bin/
chmod 0700 /usr/local/bin/mysqldump_docker_backup_schema.sh
chmod 0700 /usr/local/bin/mysqldump_docker_backup_full.sh
```

- Customize variable
```bash
vim /usr/local/bin/mysqldump_docker_backup_schema.sh
```

- Install crontab
```bash
crontab -e

# Backup mariadb with mysqldump
05  4  *  *  *  /usr/local/bin/mysqldump_docker_backup_full.sh > /dev/null 2>&1
15  4  *  *  *  /usr/local/bin/mysqldump_docker_backup_schema.sh > /dev/null 2>&1
```

- Create logrotate file
```bash
cat > /etc/logrotate.d/mysqldump_docker_backup << 'EOL'
/var/log/mysqldump_docker_backup_full.log /var/log/mysqldump_docker_backup_schema.log {
    weekly
    missingok
    rotate 4
    compress
}
EOL
```

## Database restore ![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white)

- Recovery database
```bash
# Example for docker container name `mariadb`
mkdir /tmp/recovery
tar -xvzf /var/backup/centos7/mysqldump_docker_backup_schema/kimai-mysqldump_backup_20210908_211509.sql.tar.gz -C /tmp/recovery
docker exec -i mariadb sh -c 'exec mysql -uroot -p"$MARIADB_ROOT_PASSWORD"' < /tmp/recovery/kimai-mysqldump_backup_20210908_211509.sql
```

## SMTP setup on docker host
To send mails `SMTP Server` is installed and configured on Docker host. This tutorial is suitable for `CentOS/Rocky/Oracle`.

- Install Postfix
```bash
# install postfix
dnf install postfix -y

# for sasl auth
dnf install -y cyrus-sasl cyrus-sasl-gssapi cyrus-sasl-plain
```
- Configure Postfix
```bash
# set variables
_DOMAIN=$(echo $(hostname) | cut -d"." -f2,3,4)
_HOST=$(echo $(hostname) | cut -d"." -f1)
_SMTP_SERVER=smtp.changeme.de

# create backup 
cp /etc/postfix/main.cf /etc/postfix/main.cf_orig

# configure main.cf
sed -i -e '/#myhostname = host/c\myhostname = '${_HOST}'.'${_DOMAIN}'' /etc/postfix/main.cf 
sed -i -e '/#mydomain = domain/c\mydomain = '${_DOMAIN}'' /etc/postfix/main.cf
sed -i -e 's/#myorigin = $mydomain/myorigin = $mydomain/' /etc/postfix/main.cf 
sed -i -e 's/#inet_interfaces = all/inet_interfaces = all/' /etc/postfix/main.cf
sed -i -e 's/#inet_protocols = ipv4/inet_protocols = ipv4/' /etc/postfix/main.cf 
sed -i -e '/inet_interfaces = localhost/c\inet_interfaces = \$myhostname, localhost' /etc/postfix/main.cf
sed -i -e 's/inet_interfaces = all/#inet_interfaces = all/' /etc/postfix/main.cf 
sed -i -e 's/inet_protocols = all/inet_protocols = ipv4/' /etc/postfix/main.cf
sed -i -e '/inet_protocols = ipv4/a\mynetworks = 127.0.0.0/8 172.16.0.0/16' /etc/postfix/main.cf


# Add on line 337 into main.cf after relayhost - customize smtp.changeme.de 
vim /etc/postfix/main.cf
---
# ------------ Relayhost ------------
relayhost = [smtp.changeme.de]:587
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_security_options = noanonymous
smtpd_sasl_path = smtpd
smtp_sasl_type = cyrus
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd

# ------------ Generic maps ------------
smtp_generic_maps = hash:/etc/postfix/generic
---

# create /etc/postfix/sasl_passwd file - customize "smtp.changeme.de", "helpdesk@changeme.de" and "MySuperPassword"
echo "[smtp.changeme.de]:587    helpdesk@changeme.de:MySuperPassword" > /etc/postfix/sasl_passwd
chmod 600 /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
cat /etc/postfix/sasl_passwd

# create generic maps (set domain of smtp server) - rewrite locale mail address
# customize "changeme.de"
echo "# rewrite email" >> /etc/postfix/generic
echo "root@${_HOST}.${_DOMAIN} ${_HOST}@changeme.de" >> /etc/postfix/generic
chmod 600 /etc/postfix/generic
postmap /etc/postfix/generic
cat /etc/postfix/generic

# create alias for root - customize "admin@changeme.de"
vim /etc/aliases
---
root:           admin@changeme.de
---

newaliases
```

- Run postfix
```bash
systemctl enable postfix --now
systemctl status postfix

# show running services
netstat -tulpen

# show logs
tail -f -n 2000 /var/log/maillog

# show postfix config
egrep -v '(^.*#|^$)' /etc/postfix/main.cf
```
- Create alias on smtp server `smtp.changeme.de` - customize `helpdesk@changeme.de`
```bash
${_HOST}.@${_DOMAIN}.de -> helpdesk@changeme.de
```

Enjoy!
