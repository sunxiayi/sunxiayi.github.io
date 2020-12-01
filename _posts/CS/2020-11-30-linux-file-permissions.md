---
title: Linux File Permissions
date: 2020-11-30
categories: CS
tags:
- linux
---
## Format

Use `ls -l` to see permissions, you'll see file/dir have the following format: \_rwxrwxrwx

The first character can be marked with any of the following:

- _ - no special permissions
- d - directory
- l - the file or directory is a symbolic link
- s - setuid/setgid. This will let the system run an executable as the owner with the owner's permissions
- t - sticky bit permission. This will only let the file owner rename or delete the said file.

The second group of 3 characters represent what the owner of this file can do.

The third group of 3 characters represent what the owner group of this file can do.

The fourth group of 3 characters represent what all users can do.



## Permission Groups

- owner(u)
- group(g)
- all users(a), including owner + group
- all other users(o)

Use `ls -ld file` / `stat file` to find the user & group of the file

Modify ownership:

`chmod a+rw file1` add read and write permission to file1 for all users

`chmod g-x file2` remove execution permission to file2 for group

`chmod u=rwx file1` set owner permission of file1 to be rwx

`chmod -R u=rwx file1` -R = recursively set

`chmod u+w,g+x,o-r` set multiple permissions together

`chmod u=rwx,g=rw,o=rx file` set multiple permissions together

Assign ownership: `chown owner filename` / `chgrp groupname filename`



## Permission Types

- read = 4(100)
- write = 2(010)
- execute = 1(001)

You add the numbers to get the integer/number representing the permissioons. For example:

\_rwxr\_xr\_\_ = 111101100 = (4+2+1)(4+1)(4) = 754

To change permission type for this file: `chmod 754 file`

