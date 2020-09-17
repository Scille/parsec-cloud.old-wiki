
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
# Create a directory for S3 persistent data and certificates
$ mkdir -p s3-testing
$ export S3_TESTING_DIR=$PWD/s3-testing

# Generate autosigned certificate (keys and cert)
$ openssl req -batch \
  -x509 -sha256 -nodes -days 365 -newkey rsa:4096 \
  -addext "subjectAltName = DNS:localhost" \
  -keyout $S3_TESTING_DIR/server.test.pem.key \
  -out $S3_TESTING_DIR/server.test.pem.crt

# Generate a pem file for the S3 server
$ cat $S3_TESTING_DIR/server.test.pem.key $S3_TESTING_DIR/server.test.pem.crt > $S3_TESTING_DIR/server.test.pem

# Export the certificate path for S3 client
$ export AWS_CA_BUNDLE=$S3_TESTING_DIR/server.test.pem.crt

# Run a detached postgres container called `parsec-s3`
$ docker run -d --rm \
  --name parsec-s3 \
  -e SERVICES=s3 \
  -e DATA_DIR=/tmp/localstack/data \
  -p 4566:4566 \
  -v $S3_TESTING_DIR:/tmp/localstack \
  localstack/localstack

# The docker container should now be up and running
$ docker container ls -l
CONTAINER ID   IMAGE                   COMMAND                  CREATED              STATUS              PORTS   NAMES
7f4d8467dde9   localstack/localstack   "docker-entrypoint.sh"   About a minute ago   Up About a minute   [...]   parsec-s3

# For your information, you can get containers logs using
$ docker logs s3
[...]
Running on 0.0.0.0:4566 over https (CTRL + C to quit)
Running on 0.0.0.0:38105 over http (CTRL + C to quit)

# The S3 object storage should now be exposed as an HTTPS service on port 4566
# The data is stored persistently at `$S3_TESTING_DIR/data`

# Install AWS client and setup AWS credential with dummy values
$ sudo apt update
$ sudo apt install awscli
$ aws configure
AWS Access Key ID [****************]:
AWS Secret Access Key [****************]:
Default region name [None]:
Default output format [None]:

# Create the parsec S3 bucket using the AWS client
$ aws --endpoint-url https://localhost:4566 s3 mb s3://parsec
make_bucket: parsec
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
Parsec requires SSL certificates. 

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
