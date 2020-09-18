
This guide covers the installation procedure for the parsec metadata server (usually referred to as "backend")  for **Ubuntu 18.04 LTS (Bionic)**.

1. [Preamble](#Preamble)
2. [Database requirements](#Database-requirements)
3. [Object storage requirements](#Object-storage-requirements)
4. [SSL configuration](#SSL-configuration)
5. [Parsec metadata server installation](#Parsec-metadata-server-installation)
6. [Server configuration](#Server-configuration)
7. [Start the parsec server](#Start-the-parsec-server)
8. [Create an organization](#Create-an-organization)

Preamble
--------

The parsec metadata server depends on external components in order to work properly. This includes:
- a PostgreSQL database to store the metadata
- an S3 object storage to store the data blocks
- an SMTP server for sending emails
- an SSL certificate for HTTPS communication with the clients

From a security standpoint, the installation of those components is outside the scope of this guide. In order to securely manage those services, please refer to their official documentation.

As configuring those components properly can be tedious, this guide provides instructions for quickly setting up mockups or basic install for the required services. Keep in mind that those instructions are provided for convenience and should not be used in production.

Those mockups are meant to run in docker containers. In order to install docker, please use the following commands:

```shell
# Update the list of available packages
$ sudo apt update

# Install the snap service
$ sudo apt install snapd

# Install docker through snap
$ sudo snap install docker
```

Database requirements
---------------------

The parsec metadata server requires the access to a [PostgreSQL](https://www.postgresql.org/) database, accessible through a `postgresql://` URL. This database is used to store all the metadata for the different parsec organizations. Note that the postgresql database doesn't have to run on the same machine as the metadata server.

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
CONTAINER ID   IMAGE      CREATED        STATUS        PORTS                   NAMES
03aba882daf8   postgres   2 minutes ago  Up 2 minutes  0.0.0.0:5435->5432/tcp  parsec-postgres

# For more information, show the container logs
$ docker logs parsec-postgres
[...]
2020-09-17 16:05:04.446 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2020-09-17 16:05:04.506 UTC [1] LOG:  database system is ready to accept connection
```
A dockerized postgresql service is now running with a user called `parsec`, the password being `DBPASS`. A fresh database called `parsec` has been created for the `parsec` user. The database is stored persistently at `/var/lib/docker/volumes/parsec-postgres-data`.
The postgres service is running on port 5432 in the container, but exposed on port 5435 on the local machine in order to prevent conflicts with existing services.

Now export the postgresql URL as `PARSEC_DB` and make sure the parsec database is accessible:

```shell
# Export the database URL as `PARSEC_DB`
$ export PARSEC_DB=postgresql://parsec:DBPASS@localhost:5435/parsec

# Make sure the parsec database is accessible
$ sudo apt install postgresql-client
$ psql $PARSEC_DB -c "\conninfo"
You are connected to database "parsec" as user "parsec" 
on host "localhost" (address "127.0.0.1") at port "5435"
```

Remember that those commands are provided for convenience in the case of a test environment. For more information about how to securely set up a postresql databse, please refer to [the official documentation](https://www.postgresql.org/docs/).


Object storage requirements
---------------------------

The parsec metadata server requires to access a cloud storage service to store the data. This storage is typically an [AWS S3 object storage](https://aws.amazon.com/s3/) and is used to store the blocks of encrypted data.

In the case of a test environment, an S3 storage mockup can be setup through the [localstack](https://github.com/localstack/localstack) docker container using the following commands. First generate a self-signed certificate for the SSL connection between the parsec server and the S3 mockup:
```shell
# Create a directory for S3 persistent data and certificates
$ mkdir -p s3-testing
$ export S3_TESTING_DIR=$PWD/s3-testing

# Generate self-signed certificate (keys and cert)
$ openssl req -batch \
  -x509 -sha256 -nodes -days 365 -newkey rsa:4096 \
  -addext "subjectAltName = DNS:localhost" \
  -keyout $S3_TESTING_DIR/server.test.pem.key \
  -out $S3_TESTING_DIR/server.test.pem.crt

# Generate a pem file for the S3 server
$ cat \
  $S3_TESTING_DIR/server.test.pem.key \
  $S3_TESTING_DIR/server.test.pem.crt \
  > $S3_TESTING_DIR/server.test.pem

# Export the certificate path for S3 client
$ export AWS_CA_BUNDLE=$S3_TESTING_DIR/server.test.pem.crt
```
Now run the S3 service using the [localstack](https://github.com/localstack/localstack) container:
```shell
# Run a detached localstack container called `parsec-s3`
$ docker run -d --rm \
  --name parsec-s3 \
  -e SERVICES=s3 \
  -e DATA_DIR=/tmp/localstack/data \
  -p 4566:4566 \
  -v $S3_TESTING_DIR:/tmp/localstack \
  localstack/localstack

# Check docker container is running
$ docker container ls -l
CONTAINER ID   IMAGE                   CREATED              STATUS              PORTS   NAMES
7f4d8467dde9   localstack/localstack   About a minute ago   Up About a minute   [...]   parsec-s3

# For more information, show the container logs
$ docker logs parsec-s3
[...]
Running on 0.0.0.0:4566 over https (CTRL + C to quit)
Running on 0.0.0.0:38105 over http (CTRL + C to quit)
```
The S3 object storage should now be exposed as an HTTPS service on port 4566. The data is stored persistently at `$S3_TESTING_DIR/data`. A dedicated S3 bucket called `s3://parsec` needs to be created, this can be done using the AWS client:
```shell
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
The parsec S3 bucket is now ready to be accessed using the following URL:
```shell
$ export PARSEC_BLOCKSTORE=s3:localhost\\:4566:region1:parsec:dummy-user:dummy-password
```

Again, remember that those commands are provided for convenience in the case of a test environment. For more information about how to securely set up an S3 bucket, please refer to [the official documentation](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html).


SSL configuration
-----------------

The communication between the parsec client and the parsec metadata server should be secured using SSL certificates. This can be done by either:
- Running the server behind a reverse proxy like [NGINX](http://nginx.org/en/docs), [configured as an HTTPS server](http://nginx.org/en/docs/http/configuring_https_servers.html)
- Passing a pair of SSL certificate and key explicitly to the parsec server using the `PARSEC_SSL_KEYFILE` and `PARSEC_SSL_CERTFILE` environment variables

For a test environment, self-signed SSL certificate can be generated using the following commands:
```shell

# Generate an self-signed SSL certificate
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

Remember that self-signed certificate should only be used in the context of a test environment. For more information about how to securely set up SSL certificates, please refer to the official resources of SSL certificate service providers like [Let's Encrypt](https://letsencrypt.org/).

Parsec metadata server installation
-----------------------------------

Two ubuntu packages first need to be installed through the package management system, `python3.6` and `python3-venv`:

```shell
# Update the list of available packages
$ sudo apt update

# Install Python 3.6 with venv
$ sudo apt install  python3.6 python3-venv

# Make sure python 3.6 is available
$ python3.6 -V
Python 3.6.9
```

A dedicated python [virtual env](https://docs.python.org/3/library/venv.html) then needs to be set up, using the following commands:

```shell
# Create a virtual env at ./venv
$ python3.6 -m venv venv

# Activate the freshly created venv
$ source ./venv/bin/activate

# Makes sure the python command points to the venv
$ which python
/home/user/venv/bin/python
```

The parsec metadata server is now ready to be installed through `pip`:

```shell
# Install the parsec backend specifically
$ pip install parsec-cloud[backend]==2.0.0

# Make sure the parsec backend is correctly installed
$ parsec backend --help
Usage: parsec backend [OPTIONS] COMMAND [ARGS]...
[...]
```

Server configuration
--------------------

Parsec can be configured through environment variables. 

The example bellow covers the configuration for a mockups based environment.

First configure parsec host and port. This defined which network interfaces will be used to serve the metadata server.

```shell
# Configure the parsec server bind address (default is 127.0.0.1)
$ export PARSEC_HOST=127.0.0.1

# Configure the parsec server port (default is 6777)
$ export PARSEC_PORT=6777

```

Parsec metadata server requires to be configured with external components. 
The metadata server will query those external services based on URLs.
It also possible to provide the SSL keys for the https communication.
(Those variables have already been defined in the previous sections)

```shell
# Setup the postgresql data url.
export PARSEC_DB=postgresql://parsec:DBPASS@localhost:5435/parsec

# Configure the parsec blockstore url. 
$ export PARSEC_BLOCKSTORE=s3:localhost\:4566:region1:parsec:user:password

# SSL certificate and key filenames.
$ export PARSEC_SSL_KEYFILE=$PWD/ssl-testing/parsec.test.key
$ export PARSEC_SSL_CERTFILE=$PWD/ssl-testing/parsec.test.cert
```

The way for the client to access the metadata parsec server need to be configured. It is a different option than the Parsec Host. This variable need to consider the network architecture (for example, the reverse proxy address can be setup there). This option defined the way for the client to reach the metadata server. This variable is used to compute parsec link and to generate invitation emails. The backend address is setup as a parsec url: `parsec://host:port`
```shell
# URL to reach the the parsec metadata server (used in invitation emails).
export PARSEC_BACKEND_ADDR=parsec://localhost:6677
```

In order for the administrator to perform administration operation, such as organization creation, the metadata server needs a secret administration token.
```shell
# secret administration token used to create organization
export PARSEC_ADMINISTRATION_TOKEN=s3cr3t
```


Start the parsec server
--------------------
The parsec installation is now completed. The following commands can help to assert the configuration

```shell
# Check your parsec configuration
$ env | grep PARSEC
PARSEC_HOST=localhost
PARSEC_PORT=6677
PARSEC_BLOCKSTORE=s3:localhost\:4566:region1:parsec:user:password
PARSEC_BACKEND_ADDR=parsec://localhost:6677
PARSEC_DB=postgresql://parsec:DBPASS@localhost:5435/parsec
PARSEC_ADMINISTRATION_TOKEN=s3cr3t
PARSEC_SSL_KEYFILE=/home/user/ssl-testing/parsec.test.key  # Optional
PARSEC_SSL_CERTFILE=/home/user/ssl-testing/parsec.test.cert  # Optional

# Be sure to activate the virtual env
$ source ./venv/bin/activate
```

If the postgresql database is new (or if it's the first time parsec runs), database tables need be created. This can also be run after a parsec backend update.

```shell
# Create parsec database tables
$ parsec backend migrate
Migrate ✔
0001_initial.sql ✔
0002_add_migration_table.sql ✔
0003_human_handle.sql ✔
0004_invite.sql ✔
0005_redacted_certificates.sql ✔
```


The metadata server can now be started
```shell
$ parsec backend run
Starting Parsec Backend on 127.0.0.1:6777 (db=POSTGRESQL blockstore=S3 backend_addr=parsec://localhost:6677 email_config=MockedEmailConfig(sender=no-reply@parsec.com, tmpdir=/tmp/tmp-email-folder-w_anjztu))

```


Create an organization
--------------------

Administrative operation requires the installation of the [parsec client](https://docs.parsec.cloud/en/latest/userguide/installation.html). This operation can be performed on an other machine than the 

To create an organization, the administrator need to provide:
 - The parsec metadata server location, through a parsec url `parsec://hostname:port`
 - The `administration_token` configured in the parsec metadata server.
 - An organization name


```shell
$ parsec core create_organization --help
Usage: parsec core create_organization [OPTIONS] NAME

Options:
  -B, --addr FROM_URL             [required]
  -T, --administration-token TEXT
                                  [required]
  -E, --expiration-date [%Y-%m-%d|%Y-%m-%dT%H:%M:%S|%Y-%m-%d %H:%M:%S]
  --help                          Show this message and exit
```

The following line can be used to create an organization with the mockups based environment:
```shell
$ parsec core create_organization --administration-token=s3cr3t \
     --addr=parsec://localhost:6677 TestOrganization
Creating organization in backend ✔
Bootstrap organization url: parsec://127.0.0.1:6677/TestOrganization?action=bootstrap_organization&token=79cc833ccb3d923479894578f6ebeada2681c1957aa17df03d618f121a608abf&no_ssl=true

```

This command provides a parsec url to bootstrap the organization from the parsec GUI (By pasting this url in `Join an organization` from the GUI menu`)