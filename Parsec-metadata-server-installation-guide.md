
This guide covers the installation procedure for the parsec metadata server (usually referred to as "backend")  for **Ubuntu 18.04 LTS (Bionic)**.


Database requirements
---------------------

The parsec metadata server requires the access to a [PostgreSQL](https://www.postgresql.org/), accessible through a `postgresql://` URL. This database is used to store all the metadata for the different parsec organizations. Note that the postgresql database doesn't have to run on the same machine as the metadata server.

In the case of a test environment, it might be simpler to run a postgresql database in a docker container. The following setup commands are provided for convenience:
```shell
# Run a detached postgres container called `parsec-postgres`
$ docker run -d \
  --name parsec-postgres \
  -e POSTGRES_USER=parsec \
  -e POSTGRES_PASSWORD=DBPASS \
  -e POSTGRES_DB=parsec \
  -v parsec-postgres-data:/var/lib/postgresql/data/  \
  -p 5435:5432 \
  postgres

# The container should now be up and running.
$ docker container ls -l
CONTAINER ID   IMAGE      COMMAND                  CREATED        STATUS        PORTS                   NAMES
03aba882daf8   postgres   "docker-entrypoint.s…"   2 minutes ago  Up 2 minutes  0.0.0.0:5435->5432/tcp  parsec-postgres

# It is configured with a user called `parsec`, the password being `DBPASS`.
# A fresh database called `parsec` has been created for the `parsec` user.
# The database is stored persistently at `/var/lib/docker/volumes/parsec-postgres-data`
# The postgres service is running on port 5432 in the container, but exposed on port 5435
# on the local machine in order to prevent conflicts with existing services.

# Export the database URL as `PARSEC_DB`
$ export PARSEC_DB=postgresql://parsec:DBPASS@localhost:5435/parsec

# Make sure the parsec database is accessible
$ sudo apt postgresql-client
$ psql $PARSEC_DB -c "\conninfo"
You are connected to database "parsec" as user "parsec" on host "localhost" (address "127.0.0.1") at port "5435"
```

For more information about how to securely set up a postresql databse, please refer to [the official documentation](https://www.postgresql.org/docs/).


Object storage requirements
---------------------------
The parsec metadata server requires to access a cloud storage service to store the data. This storage can be an `aws s3` storage, accessible through a s3 url.

In the case of a test environment, a s3 storage can be setup through the [localstack](https://github.com/localstack/localstack) docker container using the following commands:
```shell
# Generate autosigned certificate (keys and cert)
$ mkdir s3-ssl-cert -p
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:4096 -keyout s3-ssl-cert/server.test.pem.key -out s3-ssl-cert/server.test.pem.crt
$ cat s3-ssl-cert/server.test.pem.key s3-ssl-cert/server.test.pem.crt > s3-ssl-cert/server.test.pem
# Define cert directory for S3 server
$ export S3_SERVER_CERT_DIR=$PWD/s3-ssl-cert/
# Define cert path for S3 client
$ export AWS_CA_BUNDLE=$PWD/s3-ssl-cert/server.test.pem.crt

# Start docker service
$ docker run -p 4566:4566 -e "SERVICES=s3" --name s3 -v $S3_SERVER_CERT_DIR:/tmp/localstack -d  localstack/localstack

# Check docker container is running
$ docker container ls -l
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                                             NAMES
18bc659b7eca        localstack/localstack   "docker-entrypoint.sh"   58 seconds ago      Up 57 seconds       4567-4597/tcp, 0.0.0.0:4566->4566/tcp, 8080/tcp   s3

# For your information, you can get containers logs using 
$ docker logs s3

# Install aws client
$ sudo apt install awscli

# Setup aws credential 
$ aws configure

# Create aws s3 bucket
$ aws --endpoint-url https://localhost:4566 s3 mb s3://parsec
```

Package requirements
--------------------

Two packages need to be installed through the package management system:
 - **python3.6**
 - **python3-venv**

 See below the corresponding commands:

```shell
# Update the list of available packages
$ sudo apt update

# Install Python 3.6 with venv
$ sudo apt install  python3.6 python3-venv

# Make sure python 3.6 is available
$ python3.6 -V
Python 3.6.9
```

Setting up a python venv
------------------------

A dedicated python virtual env needs to be set up, using the following commands:

```shell
# Create a virtual env at ./venv
$ python3.6 -m venv venv

# Activate the freshly created venv
$ source ./venv/bin/activate

# Makes sure the python command points to the venv
$ which python
/home/user/venv/bin/python
```

Parsec backend installation
---------------------------

The parsec backend is now ready to be installed through `pip`:

```shell
# Install the parsec backend specifically
$ pip install parsec-cloud[backend]==2.0.0

# Make sure the parsec backend is correctly installed
$ parsec backend --help
Usage: parsec backend [OPTIONS] COMMAND [ARGS]...
[...]
```
Parsec requires SSL certificate. 

For a test environment, autosigned SSL certificate can be generated using the following commands:
```shell

# Generate an autosigned SSL certificate
$ mkdir -p ssl-testing
$ openssl req -batch \
  -x509 -sha256 -nodes -days 365 -newkey rsa:4096 \
  -keyout $PWD/ssl-testing/parsec.test.key \
  -out $PWD/ssl-testing/parsec.test.cert \
  -addext "subjectAltName = DNS:localhost"

# Export SSL certificate and key filenames for the parsec server
$ export PARSEC_SSL_KEYFILE=$PWD/ssl-testing/parsec.test.key
$ export PARSEC_SSL_CERTFILE=$PWD/ssl-testing/parsec.test.cert

# Export SSL certificate filename for the parsec client
$ export SSL_CAFILE=$PWD/ssl-testing/parsec.test.cert
```

Server configuration
--------------------

The backend can be configured through environment variables.

First configure a host and a port for the parsec server:

```shell
# Configure the parsec server bind address (default is 127.0.0.1)
$ export PARSEC_HOST=127.0.0.1

# Configure the parsec server port (default is 6777)
$ export PARSEC_PORT=6777
```

// TODO
