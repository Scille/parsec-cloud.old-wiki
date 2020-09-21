
This guide covers the installation procedure for the [parsec][parsec] metadata server [version 2.0.0](https://github.com/Scille/parsec-cloud/releases/tag/v2.0.0) for **Ubuntu 18.04 LTS (Bionic)**.

1. [Preamble](#preamble)
2. [Database requirements](#database-requirements)
3. [Object storage requirements](#object-storage-requirements)
4. [SMTP server requirements](#smtp-server-requirements)
5. [SSL configuration](#ssl-configuration)
6. [Parsec metadata server installation](#parsec-metadata-server-installation)
7. [Server configuration](#server-configuration)
8. [Start the parsec server](#start-the-parsec-server)
9. [Create an organization](#create-an-organization)


Preamble
--------

The [parsec][parsec] metadata server (also referred to as "backend") depends on external components in order to work properly. This includes:

- a [PostgreSQL][postgresql] database to store the metadata
- an [S3 object storage][s3] to store the data blocks
- an [SMTP server][smtp] for sending emails
- a [TLS/SSL server certificate][ssl] for HTTPS communication with the clients

From a security standpoint, the installation of those components is outside the scope of this guide. In order to securely manage those services, please refer to their official documentation.

As configuring those components properly can be tedious, this guide provides instructions for quickly setting up mockups or basic install for the required services. Keep in mind that those instructions are provided for convenience and should not be used in production.

Those mockups are meant to run in [docker containers][docker]. In order to [install docker](https://snapcraft.io/install/docker/ubuntu) using [snap](https://snapcraft.io/), please use the following commands:

```shell
# Update the list of available packages
$ sudo apt update

# Install the snap service
$ sudo apt install snapd

# Create docker group
$ sudo addgroup --system docker

# Add current user to the docker group
$ sudo adduser $USER docker

# Reload groups
$ newgrp docker

# Install docker through snap
$ sudo snap install docker

# Assert docker installation completed
$ docker ps
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES

```

Database requirements
---------------------

The [parsec][parsec] metadata server requires the access to a [PostgreSQL][postgresql] database, accessible through a `postgresql://` URL. This database is used to store all the metadata for the different [parsec][parsec] organizations. Note that the [postgresql][postgresql] database doesn't have to run on the same machine as the metadata server.

In the case of a test environment, it might be simpler to run a [postgresql][postgresql] database in a [docker container][docker] using the [official postgres image](https://hub.docker.com/_/postgres/). The following setup commands are provided for convenience:

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
$ docker ps -l
CONTAINER ID   IMAGE      CREATED        STATUS        PORTS                   NAMES
03aba882daf8   postgres   2 minutes ago  Up 2 minutes  0.0.0.0:5435->5432/tcp  parsec-postgres

# For more information, show the container logs
$ docker logs parsec-postgres
[...]
2020-09-17 16:05:04.446 UTC [1] LOG:  listening on IPv4 address "0.0.0.0", port 5432
2020-09-17 16:05:04.506 UTC [1] LOG:  database system is ready to accept connection
```

A dockerized [postgresql][postgresql] service is now running with a user called `parsec`, the password being `DBPASS`. A fresh database called `parsec` has been created for the `parsec` user. The database is stored persistently at `/var/lib/docker/volumes/parsec-postgres-data`.
The [postgresql][postgresql] service is running on port 5432 in the container, but exposed on port 5435 on the local machine in order to prevent conflicts with existing services.

Now export the [postgresql][postgresql] URL as `PARSEC_DB` and make sure the parsec database is accessible:

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

The [parsec][parsec] metadata server requires to access a cloud storage service to store the data. This storage is typically an [AWS S3 object storage][s3] and is used to store the blocks of encrypted data.

In the case of a test environment, an [S3 storage][s3] mockup can be setup through the [localstack](https://github.com/localstack/localstack) docker container using the following commands. First generate a [self-signed certificate][ssl] for the SSL connection between the parsec server and the S3 mockup:

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
$ docker ps -l
CONTAINER ID   IMAGE                   CREATED              STATUS              PORTS   NAMES
7f4d8467dde9   localstack/localstack   About a minute ago   Up About a minute   [...]   parsec-s3

# For more information, show the container logs
$ docker logs parsec-s3
[...]
Running on 0.0.0.0:4566 over https (CTRL + C to quit)
Running on 0.0.0.0:38105 over http (CTRL + C to quit)
```

The [S3 object storage][s3] should now be exposed as an HTTPS service on port 4566. The data is stored persistently at `$S3_TESTING_DIR/data`. A dedicated S3 bucket called `s3://parsec` needs to be created, this can be done using the AWS client:

```shell
# Install AWS client and setup AWS credential with dummy values
$ sudo apt update
$ sudo apt install awscli
$ aws configure
AWS Access Key ID [None]: default
AWS Secret Access Key [None]: default
Default region name [None]:
Default output format [None]:

# Create the parsec S3 bucket using the AWS client
$ aws --endpoint-url https://localhost:4566 s3 mb s3://parsec
make_bucket: parsec
```
The parsec S3 bucket is now ready to be accessed using the following URL:
```shell
# Note: port customization is done by escaping `:` with a `\` (e.g `localhost\\:4566`)
$ export PARSEC_BLOCKSTORE=s3:localhost\\:4566:region1:parsec:dummy-user:dummy-password
```

Again, remember that those commands are provided for convenience in the case of a test environment. For more information about how to securely set up an S3 bucket, please refer to [the official documentation](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html).


SMTP server requirements
------------------------

The [parsec][parsec] metadata server requires the access to an [SMTP server][smtp] in order to send email, when a new user is invited into an organization for instance.

For a test environment, an SMTP mockup server can be used to expose a web interface showing the emails that were meant to be sent. Here's the instructions to run [MailHog](https://github.com/mailhog/MailHog) in a [docker container][docker]:

```shell
# Run a detached localstack container called `parsec-smtp`
$ docker run -d --rm --name parsec-smtp -p 8025:8025 -p 1025:1025 mailhog/mailhog

# Check docker container is running
$ docker ps -l
CONTAINER ID   IMAGE             CREATED        STATUS         PORTS   NAMES
72b6b8ad751e   mailhog/mailhog   6 seconds ago  Up 5 seconds   [...]   parsec-smtp

# For more information, show the container logs
$ docker logs parsec-smtp
[...]
2020/09/18 14:38:27 [SMTP] Binding to address: 0.0.0.0:1025
2020/09/18 14:38:27 Serving under http://0.0.0.0:8025/
```

See below the corresponding configuration using the parsec environment variables. The mockup server can be tested by sending a dummy email using curl and displaying it through the web interface:

```shell
# Export the email server access configuration
$ export PARSEC_EMAIL_HOST=localhost
$ export PARSEC_EMAIL_PORT=1025
$ export PARSEC_EMAIL_SENDER=parsec@my-company.com
$ export PARSEC_EMAIL_HOST_USER=dummy-user
$ export PARSEC_EMAIL_HOST_PASSWORD=dummy-password

# Send a dummy email using curl
$ sudo apt update
$ sudo apt install curl
$ curl \
  --url "smtp://$PARSEC_EMAIL_HOST:$PARSEC_EMAIL_PORT" \
  --user "$PARSEC_EMAIL_HOST_USER@localhost:PARSEC_EMAIL_HOST_PASSWORD" \
  --mail-from $PARSEC_EMAIL_SENDER \
  --mail-rcpt rcpt@test.com \
  --upload-file <(echo body)

# Make sure the mail has been correctly sent to the SMTP mockup server
$ sensible-browser http://localhost:8025
```

Note that this SMTP mockup server does not run with [SSL or TLS][ssl], which is not recommended for a production server. Make sure to follow the recommendation of the official documentation for your particular [SMTP server][smtp]. In order to enable explicit or implicit TLS for the connection to the [SMTP server][smtp], please use the following parsec environment variables:

```shell
# Wether explicit TLS (typically port 587) should be used for connecting to the SMTP server
$ export PARSEC_EMAIL_USE_SSL=false  # or `true`
# Wether implicit TLS (typically port 465) should be used for connecting to the SMTP server
$ export PARSEC_EMAIL_USE_TLS=false  # or `true`
```

SSL configuration
-----------------

The communication between the parsec client and the parsec metadata server should be secured using [TLS/SSL certificates][ssl]. This can be done by either:

- Running the server behind a reverse proxy like [NGINX](http://nginx.org/en/docs), [configured as an HTTPS server](http://nginx.org/en/docs/http/configuring_https_servers.html)
- Passing a pair of [SSL certificate and key][ssl] explicitly to the parsec server using the `PARSEC_SSL_KEYFILE` and `PARSEC_SSL_CERTFILE` environment variables

For a test environment, [self-signed SSL certificate][ssl] can be generated using the following commands:

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

Remember that [self-signed certificate][ssl] should only be used in the context of a test environment. For more information about how to securely set up [SSL certificates][ssl], please refer to the official resources of SSL certificate service providers like [Let's Encrypt](https://letsencrypt.org/).

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

# Install wheel python package inside the virtual env
$ pip install wheel
```

The [parsec][parsec] metadata server is now ready to be installed through [pip](https://pip.pypa.io/en/stable/), using the `backend` switch:

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

The [parsec][parsec] metadata server can be configured through environment variables. The example bellow covers the configuration for a mockups based environment. First configure parsec host and port. This defines which network interfaces will be used to serve the metadata server.

```shell
# Configure the parsec server bind address (default is 127.0.0.1)
$ export PARSEC_HOST=127.0.0.1

# Configure the parsec server port (default is 6777)
$ export PARSEC_PORT=6777

```

The [parsec][parsec] metadata server requires to be configured with external components.
The metadata server will query those external services based on URLs.
It also possible to provide the SSL keys for the https communication
(Those variables have already been defined in the previous sections).

```shell
# Setup the postgresql data url
$ export PARSEC_DB=postgresql://parsec:DBPASS@localhost:5435/parsec

# Configure the parsec blockstore url
$ export PARSEC_BLOCKSTORE=s3:localhost\\:4566:region1:parsec:user:password

# SSL certificate and key filenames
$ export PARSEC_SSL_KEYFILE=$PWD/ssl-testing/parsec.test.key
$ export PARSEC_SSL_CERTFILE=$PWD/ssl-testing/parsec.test.cert
```

The way for the client to access the [Parsec][parsec] metadata server need to be configured. It is a different option than the parsec host. This variable need to consider the network architecture (for example, the reverse proxy address can be setup there). This option defines the way for the client to reach the metadata server. This variable is used to compute parsec link and to generate invitation emails. The server address is set up as a parsec url: `parsec://host:port`:
```shell
# URL to reach the the parsec metadata server (used in invitation emails).
export PARSEC_BACKEND_ADDR=parsec://localhost:6777
```

In order for the administrator to perform administration operation, such as organization creation, the metadata server needs a secret administration token:

```shell
# secret administration token used to create organization
export PARSEC_ADMINISTRATION_TOKEN=s3cr3t
```


Start the parsec server
--------------------

The [Parsec][parsec] metadata server installation is now completed. The following commands can help to assert the configuration:

```shell
# Check your parsec configuration
$ env | grep PARSEC
PARSEC_HOST=localhost
PARSEC_PORT=6777
PARSEC_BLOCKSTORE=s3:localhost\:4566:region1:parsec:user:password
PARSEC_BACKEND_ADDR=parsec://localhost:6777
PARSEC_DB=postgresql://parsec:DBPASS@localhost:5435/parsec
PARSEC_ADMINISTRATION_TOKEN=s3cr3t
PARSEC_SSL_KEYFILE=/home/user/ssl-testing/parsec.test.key
PARSEC_SSL_CERTFILE=/home/user/ssl-testing/parsec.test.cert
PARSEC_EMAIL_HOST=localhost
PARSEC_EMAIL_PORT=1025
PARSEC_EMAIL_SENDER=parsec@my-company.com
PARSEC_EMAIL_HOST_USER=dummy-user
PARSEC_EMAIL_HOST_PASSWORD=dummy-password
PARSEC_EMAIL_USE_SSL=false
PARSEC_EMAIL_USE_TLS=false

# Make sure that the parsec virtual env is activated
$ source ./venv/bin/activate
```

If the postgresql database is new (or if it's the first time parsec runs), the database tables need to be created. This can also be run after a parsec server update:

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

The metadata server can now be started using the `run` command:
```shell
$ parsec backend run
Starting Parsec Backend on 127.0.0.1:6777
  (db=POSTGRESQL blockstore=S3 backend_addr=parsec://localhost:6777 email_config=SmtpEmailConfig
  (sender=parsec@my-company.com, host=localhost, port=1025, use_ssl=False))
```

Create an organization
--------------------

Administrative operation requires the installation of the [parsec client](https://docs.parsec.cloud/en/latest/userguide/installation.html). This operation can be performed on an other machine than the metadata server. To create an organization, the administrator need to provide:

 - The [parsec][parsec] metadata server location, through a parsec url `parsec://hostname:port`
 - The `administration_token` configured in the [parsec][parsec] metadata server.
 - An organization name

The parsec client can be installed on the test machine from snap:

```shell
$ sudo snap install parsec --classic
```

It is needed to start a new terminal session to perform administration action on the test environment. Some variables need to be set on this fresh session. (The virtual environment used for the parsec server shall not be used there).

```shell
$ export SSL_CAFILE=$PWD/ssl-testing/parsec.test.cert
```
The client installation can be tested with the following command.

```shell
$ parsec.cli core create_organization --help
Usage: parsec.cli core create_organization [OPTIONS] NAME

Options:
[...]
```

The following line can be used to create an organization with the mockups based environment:

```shell
$ parsec.cli core create_organization --addr=parsec://localhost:6777 -T s3cr3t TestOrganization
Creating organization in backend ✔
Bootstrap organization url:
    parsec://127.0.0.1:6777/TestOrganization?action=bootstrap_organization&token=79cc833[...]

```

The organization has been successully created! All that is left to do is to send this bootstrap URL to the first user of the organization, along with instructions on how to [download](https://parsec.cloud/get-parsec) and [install](https://docs.parsec.cloud/en/latest/userguide/installation.html) the parsec client application. Once the application is installed, the bootstrap procedure can be triggered by either:

- Clicking on the `parsec://` link
- Opening the application and paste the link in the "Join Organization" dialog

Following the instructions will lead to the creation the first user on this particular device, as shown in the [parsec user documentation](https://docs.parsec.cloud/en/latest/userguide/new_organization.html), from the dialog titled "Boostrap the organization". Note that it's important for this first user to perform the boostrap operation on one of their device as the creation of the user includes the generation of a password-protected private key, stored on the device drive.

It will then be the responsability of this first user to invite their first collaborators. 




[parsec]: https://parsec.cloud/en
[docker]: https://www.docker.com/
[postgresql]: https://www.postgresql.org/
[s3]: https://aws.amazon.com/s3/
[smtp]: https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol
[ssl]: https://en.wikipedia.org/wiki/Public_key_certificate#TLS/SSL_server_certificate
