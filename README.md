# ADempiere Docker 3.0.0 Official Repository

## Image build based on:

- Linux                 : Ubuntu 20.04.3 LTS
- Java Development Kit  : eclipse-temurin:17
- Database              : PostgreSQL 15.1
- Application Server    : Jetty 10.0.7

[![Join the chat at https://gitter.im/adempiere/adempiere-docker](https://badges.gitter.im/adempiere/adempiere-docker.svg)](https://gitter.im/adempiere/adempiere-docker?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Welcome to the official repository for ADempiere Docker. This project is related
with the maintenance of an official image for ADempiere.

## Minimal Docker Requirements

To use this Docker image you must have your Docker engine release number greater
than or equal to 3.0.
To check the version of your Docker installation, open a terminal window and type the
the following command:

```
docker info
```

The expected output is as follows:

```
Client: Docker Engine - Community
 Version:    24.0.3
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.11.1
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.19.1
    Path:     /usr/libexec/docker/cli-plugins/docker-compose

Server:
 Containers: 15
  Running: 14
  Paused: 0
  Stopped: 1
 Images: 18
 Server Version: 24.0.3
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Using metacopy: false
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 3dce8eb055cbb6872793272b4f20ed16117344f8
 runc version: v1.1.7-0-g860f061
 init version: de40ad0
 Security Options:
  apparmor
  seccomp
   Profile: builtin
  cgroupns
 Kernel Version: 5.15.0-1038-oracle
 Operating System: Ubuntu 22.04.2 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 15.61GiB
 Name: live.thelinuxtrainer.com
 ID: 7de153af-02ab-4d13-90d4-342cdee9662a
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false

```

To verify the Docker version you're running on your server look at the _Server Version_ property.

### Status

* This project is at a beta stage for development environments.
* All the documentation is preliminary and subject to be changed and further improved..

### Project's structure

The adempiere-docker project follows the structure specified below

```
└─ adempiere-docker
   ├─ .env
   ├─ database.volume.yml
   ├─ database.yml    
   ├─ adempiere.yml
   ├─ adempiere-last
   ├─ tenant1
   |  ├─ .env
   |  ├─ Adempiere_394LTS.tar.gz
   |  ├─ lib
   |  └─ packages
   └─ tenant2
   |  ├─ .env   
   |  ├─ Adempiere_394LTS.tar.gz
   |  ├─ lib
   |  └─ packages
   ...
   └─ tenantN
   ...
```
#### .env file

This file contains the setting variables for Tenant deployment

```
ADEMPIERE_DB_PORT=55432
ADEMPIERE_DB_PASSWORD=adempiere
ADEMPIERE_DB_ADMIN_PASSWORD=postgres
```
tenant/.env
```
ADEMPIERE_WEB_PORT=8277
ADEMPIERE_SSL_PORT=4444
ADEMPIERE_VERSION=3.9.4
# ATENTION If is "Y" it will be replace the actually defined database with an empty ADempiere seed
ADEMPIERE_DB_INIT=Y 

```

#### tenant directory

This directory contains the files needed to deploy and start a particular ADempiere instance of a tenant.
Here we will find:
* The Adempiere tar.gz installer,  if not exist then will be downloaded from the last stable version.
* lib: The files to copy to the lib directory on ADempiere (this directory will contain the customization and zk-customization of ADempiere.
* packages: The files to copy to the packages directory on ADempiere (this directory will contain the localization of an ADempiere).


### database.volume.yml

this file will contain an external database volume 

```
version: '2.0'
services:
  database:
    volumes:
      - database:/var/lib/postgresql/data/
volumes:
  database:
    driver: local
```

### database.yml

this file will contain the PostgreSQL deployment

```
version: '2.0'
services:
  database:
    image: postgres:15.1
    ports:
      - "${ADEMPIERE_DB_PORT}:5432"
    volumes:
      - ./database:/var/lib/postgresql/data
    networks:
      - custom
    environment:
      - POSTGRES_USER:postgres
      - POSTGRES_PASSWORD:postgres
      - PGDATA:/var/lib/postgresql/data/pgdata
      - POSTGRES_INITDB_ARGS:''
      - POSTGRES_INITDB_XLOGDIR:''

volumes:
 database:
  external: true

networks:
  custom:
    external: true   
```      

### adempiere.yml

This file will contain the definition of our ADempiere clients.
For a client, we will need to complete the next parametrization.

```
version: '2.0'
services:
  adempiere-tenant:
    networks:
      - custom
    external_links:
      - database: database
    image: "${COMPOSE_PROJECT_NAME}" # Name of the instance for docker create based on the project name
    container_name: "${COMPOSE_PROJECT_NAME}" # Name of the ADempiere client container
    ports:
      - ${ADEMPIERE_WEB_PORT}:8888 # http port where the web client will be exposed
      - ${ADEMPIERE_SSL_PORT}:444 # https port where the web client will be exposed
    environment:
      ADEMPIERE_DB_INIT: ${ADEMPIERE_DB_INIT} # ATENTION If is "Y" it will be replace de actual defined database with a empty ADempiere seed
    build:
      context: .
      dockerfile: ./adempiere-last/Dockerfile
      args:
        ADEMPIERE_BINARY : ${ADEMPIERE_BINARY}
        ADEMPIERE_SRC_DIR: "./${COMPOSE_PROJECT_NAME}" # Directory that contains the ADempiere installer, customization, and localization
        ADEMPIERE_DB_HOST: "database"
        ADEMPIERE_DB_PORT: 5432
        ADEMPIERE_DB_NAME: "${COMPOSE_PROJECT_NAME}"
        ADEMPIERE_DB_USER: "${COMPOSE_PROJECT_NAME}"
        ADEMPIERE_DB_PASSWORD: ${ADEMPIERE_DB_PASSWORD}
        ADEMPIERE_DB_ADMIN_PASSWORD: ${ADEMPIERE_DB_ADMIN_PASSWORD}
networks:
  custom:
    external: true
```

### Postgres Container
If you don't have an external database server, You can use the Postgres server container defined in this composer. As you will not have a database defined in the container, you can first start the database container to mount it, or you can pass the ADEMPIERE_DB_INIT argument with "Y" to load an ADempiere seed, then you only need to parametrize your ADempiere instances with this database configuration.

### Usage

Edit and define the parameters for the new instance

```
nano .env 
nano ./eevolution/.env
```

to do this in the terminal we will run the next line:

note: evolution is the name for a new tenant

```
./application.sh eevolution up -d 
```


This command will build the images defined in the .env, create the containers and start them. The "-d" parameter will launch the process in the background.
To stop the containers you will run the next command.
```
./application.sh eevolution stop
```
Note that in the above command, we use the instruction ```stop'' instead of ```down```, this is because the ```down``` instruction deletes the containers to, ```stop``` only shutdown them.

If you have a new tenant, you only need to edit and setting the tenant definition to tenant/.env file and start up only this image and container.

```
./application.sh eevolution up -d 
```

If you need a backup from a Database using 

Generate backup : 

```
./application.sh eevolution exec adempiere-tenant /opt/Adempiere/utils/RUN_DBExport.sh
```
Get backup zip :

```
./application.sh eevolution exec adempiere-tenant "cat /opt/Adempiere/data/ExpDat.dmp" \ 
| gzip >  "backup.$(date +%F_%R).gz"
```


If you're not familiar with docker-compose and how to manage Docker services through docker-compose have a
look at the [docker compose documentation](https://docs.docker.com/compose)

### Contribution

Contributions are more than welcome. Please log any issue or new feature request in
adempiere-docker project's repository
