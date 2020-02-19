## Excel/Word or another Office software says the file name is too long

> This error message occurs when you save or open a file if the path to the file (including the file name) exceeds 218 characters. This limitation includes three characters representing the drive, the characters in folder names, the backslash character between folders, and the characters in the file name.

https://support.microsoft.com/en-us/help/213983/error-message-when-you-open-or-save-a-file-in-microsoft-excel-filename

Nothing Parsec can do about that.

## Too many old devices appear in the device selection dropdown

This problem usually occurs for users that tried the beta versions of Parsec with a test database that could be reset at any time, leaving some old devices still registered on their computer. We don't want Parsec to have a "clean all" functionality, as this could prove very dangerous and erase devices that were not meant to be erased.

A user can still erase these devices manually though:
**On Windows**, in the explorer's address bar, go to "%APPDATA%/parsec" (or C:\Users\<YourUser>\AppData\Roaming, AppData is a hidden folder).
**On Linux**, in your home directory (~), go to .config/parsec.

- Remove unwanted devices in config/devices
- Remove unwanted devices in data

:warning: | Be very careful. Deleting a device virtually means deleting the data stored in that device (since there's no way to decrypt them).
------------ | -------------
