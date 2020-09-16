
This guide covers the parsec backend installation procedure for **Ubuntu 18.04 LTS (Bionic)**.


Database requirements
---------------------

The parsec backend requires the access to a postresql database, accessible through a `postgresql://` URL. Note that this database doesn't have to run on the same machine as the backend.

In the case of a test environement, the following setup commands are provided for convenience. First install PostgreSQL (version 10 or above):
```shell
$ sudo apt install postgresql-10
```

Then create a PostgreSQL database called `parsec_testing` using `psql`:
```shell
$ sudo -u postgres psql
postgres=# create database parsec_testing;
postgres=# create user testing with encrypted password 'test';
postgres=# grant all privileges on database parsec-testing to testing;
postgres=# exit
```

Finally make sure that the `parsec_testing` database is available through the corresponding URL:
```shell
# Export the database URL as `PARSEC_DB`
$ export PARSEC_DB=postgresql://testing:test@localhost/parsec_testing

# Make sure the database is available
$ psql $PARSEC_DB -c "\conninfo"
```

For more information about how to securely set up a postresql databse, please refer to [the official documentation](https://www.postgresql.org/docs/).


Object storage requirements
---------------------------

// TODO


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
