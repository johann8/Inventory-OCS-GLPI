## GLPI Docker Image
Image is based on [Alpine 3.17](https://hub.docker.com/repository/docker/johann8/alpine-glpi/general)

| pull | size | version | platform |
|:---------------------------------:|:----------------------------------:|:--------------------------------:|:--------------------------------:|
| ![Docker Pulls](https://img.shields.io/docker/pulls/johann8/alpine-glpi?style=flat-square) | ![Docker Image Size (latest by date)](https://img.shields.io/docker/image-size/johann8/alpine-glpi/latest) | [![](https://img.shields.io/docker/v/johann8/alpine-glpi?sort=date)](https://hub.docker.com/r/johann8/alpine-glpi/tags "Version badge") | ![](https://img.shields.io/badge/platform-amd64-blue "Platform badge") |

## OCS Inventoryi NG Docker Image 
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

- [Install OCS Inventory](#install-ocs-inventory)
  - [OCS Inventory setup](#ocs-inventory-setup)
- [Install GLPI](#install-glpi)
  - [GLPI setup](#glpi-setup)


## Install GLPI plus OCS Inventory docker container
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
- Run through the installation wizard and log in with admin / admin

### Install GLPI
- Go to http://glpi.changme.de
- Enter the database connection details as shown in the picture
![Connect to Database](https://raw.githubusercontent.com/johann8/alpine-glpi/master/docs/assets/screenshots/GLPI_Setup_01.PNG)
- Choose the database glpi
![Choose Database](https://raw.githubusercontent.com/johann8/alpine-glpi/master/docs/assets/screenshots/GLPI_Setup_02.PNG)

- Run through the installation wizard and log in with glpi / glpi

### OCS Inventory setup
- Setup of `OCS Inventory` can be found [here](https://github.com/johann8/ocs-alpine)

### GLPI setup
- Setup of `GLPI` can be found [here](https://github.com/johann8/alpine-glpi)

Enjoy!
