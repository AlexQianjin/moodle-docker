# How To Install Moodle via git with Postgres, Nginx and Php-fpm docker images on Ubuntu 18.04
Install Moodle by docker with nginx, php-fpm and postgres images. And also using LetsEncrypt for HTTPS on Ubuntu 18.04.

## Getting the certificates from LetsEncrypt by Certbot
### Prerequisites
- An A record with el.alexqin.cn pointing to the server’s public IP address.

### Step 1 — Installing Certbot
```
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install certbot
```

### Step 2 — Running Certbot
```
sudo ufw allow 80
sudo certbot certonly --standalone --preferred-challenges http -d el.alexqin.cn
```

### Step 3 - Listing Certificates
```
sudo chmod -R 755 /etc/letsencrypt/live/el.alexqin.cn
sudo ls /etc/letsencrypt/live/el.alexqin.cn
```

## Installing Git and Docker
### Step 1 - Installing Git
```
sudo add-apt-repository ppa:git-core/ppa 
sudo apt update
sudo apt install -y git
```

### Step 2 - Getting this repository
```
cd ~/
git clone https://github.com/AlexQianjin/moodle-docker.git
```

### Step 3 - Installing docker
```
cd ~/
sudo sh moodle-docker/install/docker.sh
```

## Setting up the database
### Step 1 - Creating docker network
```
docker network create moodle-net
```

### Step 2 - Running postgres container
```
cd ~/
docker run --network moodle-net --name moodle-postgres -p 5432:5432 -e POSTGRES_PASSWORD={yourpassword} -d postgres
```

### Step 3 - Creating moodle database
```
cd ~/
sudo docker exec -it moodle-postgres /bin/bash
psql -U postgres -h localhost
postgres=# CREATE USER moodleuser WITH PASSWORD '{yourpassword}';
postgres=# CREATE DATABASE moodle WITH OWNER moodleuser;
```

## Getting the Moodle and updating the config
### Step 1 - Getting the moodle
```
cd ~/
git clone -b MOODLE_38_STABLE git://git.moodle.org/moodle.git 
```

### Step 2 - Updating the config
```
cd ~/
chmod -R 0755 /path/to/moodle
cd moodle
cp config-dis.php config.php
nano config.php
```
- Update the follow parts
```
<?PHP
unset($CFG);
global $CFG;
$CFG = new stdClass();
$CFG->dbtype    = 'pgsql';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'php:9000';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'yourpassword';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array(
    'dbpersist' => false,
    'dbsocket'  => false,
    'dbport'    => '',   
);
$CFG->wwwroot   = 'https://eq.alexqin.cn';
$CFG->dataroot  = '/usr/local/moodledata';
$CFG->directorypermissions = 02777;
$CFG->admin = 'admin';
require_once(dirname(__FILE__) . '/lib/setup.php');
```

### Step 3 - Creating the (moodledata) data directory
```
cd ~/
mkdir /usr/local/moodledata
chmod 0777 /usr/local/moodledata
```

## Setting up the php-fpm
### Step 1 - Building the php-fpm image from Dockerfile
```
cd ~/
cd moodle-docker/php/
docker build -t moodle/php:7.4-fpm ./
```

### Step 2 - Running the moodle/php:7.4-fpm container
```
cd ~/
docker run --network moodle-net -d -p 9000:9000 --name php -v $(pwd)/moodle:/var/www/html -v /usr/local/moodledata:/usr/local/moodledata moodle/php:7.4-fpm
```

## Setting up the nginx
### Step 1 - Updating the config file for nginx
```
cd ~/
nano moodle-docker/nginx/site.conf
```

### Step 2 - Running the nginx container
```
cd ~/
docker run --network moodle-net -d -p 80:80 -p 443:443 --name web -v $(pwd)/moodle:/var/www/html -v $(pwd)/moodler-docker/nginx/conf.d/:/etc/nginx/conf.d/ -v /usr/local/moodledata:/usr/local/moodledata nginx:alpine
```

## Installing the moodle in the browser
### Finishing the setting up in the browser

## pgAdmin4 Container
### Running pgAdmin4 container for Postgres
```
docker pull dpage/pgadmin4
docker run -p 8080:80 \
    --network moodle-net \
    --name pgadmin4 \
    -e 'PGADMIN_DEFAULT_EMAIL=user@domain.com' \
    -e 'PGADMIN_DEFAULT_PASSWORD=SuperSecret' \
    -d dpage/pgadmin4
```