Requirements
------------

- Python == 3.9


Installation
------------

Install conda: https://docs.conda.io/en/latest/miniconda.html
```console
% conda create -n parsec python=3.9
% conda activate parsec
```

From pypi:

```console
% poetry install parsec[all]
% pip3 install parsec[core] # Core only
% pip3 install parsec[backend] # Backend only
```

From sources:

```console
% git clone https://github.com/Scille/parsec-cloud.git
% cd parsec-cloud
% poetry install -E core -E backend
% python setup.py generate_pyqt
```

Unit testing
------------

Using pytest:

```console
% pytest                  # Quick run
% pytest --runmountpoint  # Include mountpoint tests
% pytest --rungui         # Include GUI tests
% pytest --runslow        # Include slow tests
% pytest --runrust        # Include rust tests
```

Install `pytest-xvfb` to hide the Qt windows when running the GUI tests
```
% apt install xvfb
% pip install pytest-xvfb
```

Backend
-------

Running the backend without a PostgreSQL database:

```console
% parsec backend run -b MOCKED -l INFO
Starting Parsec Backend on 127.0.0.1:6777 (db=MOCKED, blockstore=MOCKED)
```

Core
----

First step is to create an organization:

```console
% parsec core create_organization -B ws://localhost:6777 -T CCDCC27B6108438D99EF8AF5E847C3BB vcorp
Creating organization in backend ✔
Bootstrap organization url: ws://localhost:6777/vcorp?bootstrap-token=8e0c869b206feaf82eb5568533535d902486d4f3d789f47fdb3ce61735fee2a7
```

The generated URL is given to the first user to boostrap the organization.

```console
% parsec core bootstrap_organization -B "ws://localhost:6777/vcorp?bootstrap-token=8e0c869b206feaf82eb5568533535d902486d4f3d789f47fdb3ce61735fee2a7" billy@laptop
Password:
Repeat for confirmation:
Creating locally billy@laptop ✔
Sending billy@laptop to server ✔
Organization url: ws://localhost:6777/vcorp?rvk=CA4XUEKH6V2YTUG5GJWKXSU3F7GDHHT7O4KCATV4ZWNGR34NX6WAssss
```

In this example:
 - `billy` is the user id
 - `laptop` is the device name
 - `billy@laptop` is the device id

User and device private keys have been created and stored in `~/.config/parsec/devices/billy@laptop/billy@laptop.keys`:

```console
% ls ~/.config/parsec/devices/vcorp\#billy@laptop
vcorp#billy@laptop.keys
```

The root of a user drive can only be populated with workspaces, created with the following command:

```console
% parsec core create_workspace -D vcorp:billy@laptop hello
```

The user drive can now be mounted on the system.

```console
% parsec core run -D vcorp:billy@laptop
password:
billy@laptop's drive mounted at ~/parsec_mnt/billy@laptop
```

Note that only a single core context should run at a time for a given device. This means the command above needs to be stopped before running another command or using the GUI as `billy@laptop`.

Browsing the mount point:

```console
% cd ~/parsec_mnt/billy@laptop  # Or
% nautilus ~/parsec_mnt/billy@laptop  # Or
% nemo ~/parsec_mnt/billy@laptop
```

Creating a file:

```console
% echo ':)' > ~/parsec_mnt/billy@laptop/hello/world.txt
```

Now stop the `parsec core run` command and invite a new device, `pc`:

```console
% parsec core invite_device -D vcorp:billy@laptop pc
password:
Backend url: ws://localhost:6777/vcorp?rvk=CA4XUEKH6V2YTUG5GJWKXSU3F7GDHHT7O4KCATV4ZWNGR34NX6WAssss
Invitation token: 100eed5f187d28c8
Waiting for invitation reply ...
```

Separately, claim the device using the provided organization URL:


```console
% parsec core claim_device -B "ws://localhost:6777/vcorp?rvk=CA4XUEKH6V2YTUG5GJWKXSU3F7GDHHT7O4KCATV4ZWNGR34NX6WAssss" --token 100eed5f187d28c8 billy@pc
Password:
Repeat for confirmation:
Waiting for referee to reply ✔
Saving locally billy@pc ✔
```

The invite command should be finished and the `billy@laptop` device can be restarted (using `parsec core run -D billy@laptop`).

Now mount the new user drive:

```console
% parsec core run -D vcorp:billy@pc
password:
billy@pc's drive mounted at ~/parsec_mnt/billy@pc
```

And browse the new previously created file:

```console
% cat ~/parsec_mnt/billy@pc/hello/world.txt
:)
```

Note that it takes a couple of seconds before a change in given device is propagated to another one:

```console
% echo ':D' >  ~/parsec_mnt/billy@pc/hello/world.txt
% # Laptop hasn't been updated yet
% cat ~/parsec_mnt/billy@laptop/hello/world.txt
:)
% # Laptop is now updated!
% cat ~/parsec_mnt/billy@laptop/hello/world.txt
:D
```

GUI
---

Make sure that no `parsec core run` command is running and then use:

```console
% parsec core gui
```
