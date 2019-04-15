*this is a work in progress - what's presented here does not correspond to the current state of the code base*

Several modules can affect the file system of a given workspace:

- the fuse mountpoint
- the winfsp mountoint
- the path interface
- the synchronization module

Each of this modules relies on the transactions provided by the workspace file system. Those transactions have the following properties:
- **Atomicity**: they either fail or succeed, but do not leave the system in an inconsistent or transitional state
- **Safety**: they are protected internally so several transactions can be executed concurrently
- **Locality**: they only affect a single entry of the file system (more specifically, a single manifest)
- **Agnostic**: they behave the same regardless of the current platform

List of file system transactions
--------------------------------

The following transactions target entries represented by their file system path:

#### Information:
* **get_info**: returns a dictionary with all the information about the given entry

#### Rename entry:
* **rename (overwrite=True)**: rename the given entry, possibly overwriting its destination
* **rename (overwrite=False)**: rename the given entry, failing if the destination already exists

*Note 1: it is not possible to rename to another directory, as this would break the locality principle*

*Note 2: these operations affects the parent of the entry to rename and not the entry itself*

#### Folder entry:
* **folder_create**: create a new folder corresponding to the given entry
* **folder_delete**: delete the given folder corresponding to the given entry, failing if it is non-empty

*Note 3: these operations affects the parent of the entry to create/delete and not the entry itself*

#### File entry:
* **file_create**: create a new file corresponding to the given entry and returns an open file descriptor
* **file_open**: open the file corresponding to the given entry and returns a file descriptor
* **file_delete**: delete the file corresponding to the given entry

*Note 4: these operations affects the parent of the entry to create/delete and not the entry itself*

#### Synchronization:

* **file_sync**: synchronize the file corresponding to the given entry with the remote
* **folder_sync**: synchronize the folder corresponding to the given entry with the remote

*Note 5: these operations do not solve non-trivial conflicts, as it might require non-local changes. It is instead the responsibility of the synchronization module to catch these errors and deal with them.*

The following transactions target an open file represented by its file descriptor: 

#### File descriptor:
* **fd_read**: read data from the given file descriptor and **move its cursor**
* **fd_write**: write data to the given file descriptor and **move its cursor**
* **fd_resize**: change the size of a file descriptor, padding with `\x00` if necessary
* **fd_flush**: currently a no-op operation, since data 
* **fd_close**: close the file descriptor

All the file system transactions are expected to raise the standard [python OS errors](https://docs.python.org/3.7/library/exceptions.html#os-exceptions) corresponding to the encountered error.

FUSE operations
---------------

FUSE operations must specifically behave as described in [fuse_lowlevel.h](https://github.com/libfuse/libfuse/blob/fuse_2_9_5/include/fuse_lowlevel.h). They're bound to a single file system transaction:

#### Info:

* **getattr** - uses `get_info`
* **readdir** - uses `get_info`

#### Move entry:
* **rename** - uses `move(overwrite=True)`

#### Entry:
* **mkdir** - uses `folder_create`
* **rmdir** - uses `folder_delete`

#### File entry:
* **create** - uses `file_create`
* **open** - uses `file_open`
* **unlink** - uses `file_delete`

#### File descriptor:
* **read** - uses `fd_read`
* **write** - uses `fd_write`
* **truncate** - uses `fd_resize`
* **flush/fsync** - uses `fd_flush`
* **release** - uses `fd_close`

Winfsp operations
-----------------

Winfsp operations must specifically behave as described in [winfsp.h](https://github.com/billziss-gh/winfsp/wiki/WinFsp-API-winfsp.h). They're bound to a single file system transaction, except for `close` that might run both `fd_close` et `file_delete` in some cases.

#### Info:
* **get_security_by_name** - uses `get_info`
* **get_file_info** - uses `get_info`
* **can_delete** - uses `get_info`
* **read_directory** - uses `get_info`

#### Rename entry:
* **rename** - uses `move(overwrite=False)`

#### Folder Entry:
* **create** - uses `folder_create`
* **close** - uses `folder_delete`

#### File Entry:
* **create** - uses `file_create`
* **open** - uses `open`
* **close** - uses `delete`

#### File descriptor:
* **read** - uses `fd_read`
* **write** - uses `fd_write`
* **set_file_size** - uses `fd_resize`
* **flush** - uses `fd_flush`
* **close** - uses `fd_close`

Workspace path interface
------------------------

This interface is used to access a workspace file system from other parts of the application. It mimics the [pathlib.Path method interface](https://docs.python.org/3.7/library/pathlib.html#methods) for both naming and behavior.

#### Info:
- **stat** - uses `get_info`
- **is_dir** - uses `get_info`
- **is_file** - uses `get_info`

#### Rename:
- **rename** - uses `rename`
- **move** - uses `rename` or a recursive copy and delete if necessary

Note 6: `move` actually does not exist as a method of `pathlib.Path`, so it should mimic [shutil.move](https://docs.python.org/3/library/shutil.html#shutil.move) instead.

#### Folder:
- **mkdir** - uses `folder_create`
- **rmdir** - uses `folder_delete`

#### File:
- **touch** - uses `file_create` and `fd_close`
- **unlink** - uses `file_delete`
- **read_text/bytes** - uses `file_open`, `fd_read` and `fd_close`
- **write_text/bytes** - uses `file_create/file_open`, `fd_write` and `fd_close`
