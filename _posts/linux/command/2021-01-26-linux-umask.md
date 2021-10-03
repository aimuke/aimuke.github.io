---
title: "Permissions masking with umask, chmod, 777 octal permissions"
tags: [linux, umask, chmod, permission]
list_num: n
---

origin url [Permissions masking with umask, chmod, 777 octal permissions](http://teaching.idallen.com/cst8207/12f/notes/510_umask.html)

Ian! D. Allen – [idallen@idallen.ca](mailto:idallen@idallen.ca) – [www.idallen.com](http://www.idallen.com/)

# 1 `umask` blocks default permissions

When a process creates a new file system object, such as a file or directory, the permissions of that new object are determined by the **default** permissions for the object, masked by the permissions set in a process `umask`.

The process `umask` controls what permissions are _not_ given to newly created file system objects such as files and directories. It does not affect the permissions of existing objects, only of objects newly created by that process.

The `umask` also influences the `chmod` command as well as the permission of newly-created files and directories.

# 2 Default Permissions: directory `777`, file `666`

When a process creates a new file system object, such as a file or directory, the object is assigned a set of **default** permissions that is masked by the `umask`.

The default Unix permission set for newly created directories is `777` \(`rwxrwxrwx`\) masked \(blocked\) by the permission bits set in the `umask` of the process. \(See below for an explanation of Unix numeric permissions `777`.\)

The default permissions for newly created files is `666` \(`rw-rw-rw-`\) masked by the permission bits set in the `umask` of the process.

The `umask` controls what permissions are _not_ given to newly created file system object. Without the `umask`, all new directories would be created with full `777` permissions, and all new files would be created with full `666` permissions. The `umask` blocks certain permissions from being given to newly created file system objects.

# 3 Masking is not subtracting

Every bit set in the `umask` for the process “masks”, or “takes away”, that permission from the default permissions for newly created file system objects created by that process. The `umask` value is a _mask_; it turns _off_ permissions so that they are not assigned to newly created objects.

“Mask” does not mean “subtract”, in the arithmetic sense – there is no borrow or carry involved. The two binary bits `10` masked by the two bits `01` result in the two bits `10`. \(The mask `01` turns off the rightmost bit; but, it was already off, so no change.\) The two bits `10` masked by the two bits `11` result in the two bits `00`. \(The mask `11` turns off both bits.\)

The `umask` is a _mask_; it is _not_ a number to be subtracted. It turns off permissions that would normally be granted. Masking is not the same as subtracting, e.g. `666` masked with `001` is still `666` and `666` masked with `003` is `664`. The mask turns off permission bits so that they are not assigned to newly created objects. If they are already off, the `umask` makes no change:

```bash
rw- (6) masked with --x (1) is rw- (6)
  - because the x bit in the mask does not change any permissions
rw- (6) masked with -wx (3) is r-- (4)
  - because only the w bit is changed (turned off) by the mask
```

# 4 The `umask` command affects default permissions

> Many modern Linux shells also accept symbolic `umask` permissions, in addition to the traditional octal numbers. See the manual page for your shell for details. In this course, we use the traditional octal numbers that work everywhere.

The shell command `umask 022` sets to `022` \(`----w--w-`\) the permissions to be removed \(masked\) from the default permissions, for new files and directories created by the shell \(and by commands run from that shell\). It prevents write permission being assigned to group and other on newly created directories and files. A new directory would have permissions `777` \(`rwxrwxrwx`\) masked by `022` \(`----w--w-`\) resulting in `755` \(`rwxr-xr-x`\) permissions. A new file would have permissions `666` \(`rw-rw-rw-`\) masked by `022` \(`----w--w-`\) resulting in `644` \(`rw-r--r--`\) permissions.

The `umask` only applies to the permissions given to _newly created_ files and directories.

The traditional friendly Unix `umask` is `022`, resulting in default file permissions of `644` and default directory permissions of `755`. \(Newly-created files and directories are readable by anyone; but, they are only writable by the owner.\) A “secure” `umask` would be `077`. \(Mask out all group and other permissions; newly-created files and directories are readable/writable/executable only by the single user that created them.\)

The `umask` command cannot affect the permission of already-existing files. To do that, you must use the `chmod` command:

```bash
$ chmod 711 program
$ chmod go-rwx secrets
```

Note that the `umask` and the permissions assigned by `chmod` are opposites. The `chmod` command sets permissions to be given to an object; the `umask` sets permissions _not_ to be given to new objects.

Look for `umask` in some of the following pages for more examples:

* [http://www.ucolick.org/~ksa/manual/level2.html\#umask](http://www.ucolick.org/~ksa/manual/level2.html#umask)
* [http://www.cis.rit.edu/class/simg211/unixintro/Access\_Permissions.html](http://www.cis.rit.edu/class/simg211/unixintro/Access_Permissions.html)
* [http://www.uvm.edu/~hag/wcreate/644.html](http://www.uvm.edu/~hag/wcreate/644.html)

# 5 `umask` is set and then passed to child processes

Every process on Unix \(including every shell process\) has its own `umask` value, which can be changed. The system standard `umask` is set for you at login and is inherited by child processes.

Different Linux distributions set different standard \(at login\) `umask` values. The values in your particular distribution of Linux may not be the same as other distributions. The values set by the system administrator may differ from the distribution defaults. Do not rely on the `umask` having any standard value.

Every shell script should set `umask` at the beginning, so that files and directories created by the script \(and by child processes of the script\) have known permissions.

# 6 `umask` affects `chmod`

Using the `chmod` command without specifying whether you want to change User, Group, or Other permissions \(e.g. `chmod +x foo`\) causes `chmod` to use your `umask` to decide what sets of permissions to change. The `umask` setting causes `chmod` to ignore changes for the masked permissions. For example:

```bash
umask 0011 ; chmod +x foo   # only adds User x permissions
umask 0111 ; chmod +x foo   # does nothing (no permissions changed)
umask 0400 ; chmod -r foo   # only removes Group and Other r permissions
umask 0444 ; chmod -r foo   # does nothing (no permissions changed)
umask 0727 ; chmod +rwx foo # adds only Group rx permissions
```

The `umask` value tells `chmod` which permissions `chmod` is allowed to affect. The masked-out permissions are not affected. If you want `chmod` to ignore the current `umask`, specify exactly which permission sets to affect:

```bash
umask 0077 ; chmod g+x foo   # ignores umask; adds Group x permissions
umask 0700 ; chmod u-r foo   # ignores umask; removes User r permissions
```

Always specify the precise User/Group/Other permission string when using `chmod`, since you don’t know what the current `umask` might be.

# 7 Using numeric `022`-style octal permissions

Unix permissions for user, group, and other have traditionally been expressed using a set of three \(octal\) digits, where each digit represents the octal number you get by expressing the three `rwx` permissions in binary form. Convert the enabled permission bits in `rwx` into binary, using `1` for enabled and `0` for not enabled, then convert the binary number to an octal digit. Three sets of three permissions becomes three \(octal\) digits, e.g. `rwxr-x-wx` becomes `111|101|011` which is`753`.

Permissions \(mode\) can be represented in two ways: symbolic \(three letters\) or numeric \(one octal digit\). The single octal digit represents the three symbolic letters using a numeric weighting scheme shown below. The permission is treated as a binary number, with zeroes taking the place of the dashes \(not enabled\) and ones taking the place of the allowed permissions.

Numeric weighting for each of the three `rwx` permissions \(three binary digits to one octal digit\):

```bash
x (execute) --x  becomes 001 binary and has binary weight 2^0 = 1
w (write)   -w-  becomes 010 binary and has binary weight 2^1 = 2
r (read)    r--  becomes 100 binary and has binary weight 2^2 = 4
```

Each of the three sets of symbolic permissions \(user/owner, group, other\) can be summarized by a single octal digit by adding up the three numeric `rwx` values using the three weights \(4,2,1\) given above:

```bash
rwx ==> 111 binary ==> digit 7 because r is 4, w is 2, and x is 1 so 4+2+1=7
r-x ==> 101 binary ==> digit 5 because r is 4 and x is 1 so          4+0+1=5
-wx ==> 011 binary ==> digit 3 because w is 2 and x is 1 so          0+2+1=3
--- ==> 000 binary ==> digit 0 because no permissions are set so     0+0+0=0

7 = binary 111 = rwx
6 = binary 110 = rw-
5 = binary 101 = r-x
4 = binary 100 = r--
3 = binary 011 = -wx
2 = binary 010 = -w-
1 = binary 001 = --x
0 = binary 000 = ---
```

The full set of nine permission characters can then be grouped and summarized as three octal digits:

```bash
rwxr-x-wx  is  rwx|r-x|-wx  is 111|101|011 ==> the three digits 753
---r----x  is  ---|r--|--x  is 000|100|001 ==> the three digits 041
---------  is  ---|---|---  is 000|000|000 ==> the three digits 000
rwxrwxrwx  is  rwx|rwx|rwx  is 111|111|111 ==> the three digits 777
```

Make sure you always write exactly nine characters when writing symbolic permissions. Exactly nine. Do not include the leading “inode type” character when listing the nine characters of symbolic permissions.

Thus `chmod 741 file` means “set the mode to `741` \(`rwxr----x`\)”. That is `7` \(`7=111="rwx"`\) for owner, `4` \(`4=100="r--"`\) for group, and `1` \(`1=001="--x"`\) for others. In most modern Unix systems, you can do the same thing using symbolic permissions as `chmod u=rwx,g=r,o=x file`.

The shell command `umask 027` means “mask \(remove\) permissions `027` from newly created files and directories”:

* Octal `027 = "----w-rwx"` which can be split into three parts: `0=000="---"` for owner, `2=010="-w-"` for group, and `7=111="rwx"` for others.
* A new directory created under this `umask 027` \(e.g. by `mkdir`\) would have directory default permissions `777` masked by `027` = `750` \(`rwxr-x---`\).
* A new file created under this `umask 027` \(e.g. created by output redirection or by a file copy\) would have file default permissions `666` masked by `027` = `640` \(`rw-r-----`\).

```text
Author: 
| Ian! D. Allen, BA, MMath  -  idallen@idallen.ca  -  Ottawa, Ontario, Canada
| Home Page: http://idallen.com/   Contact Improv: http://contactimprov.ca/
| College professor (Free/Libre GNU+Linux) at: http://teaching.idallen.com/
| Defend digital freedom:  http://eff.org/  and have fun:  http://fools.ca/
```

[Plain Text](http://teaching.idallen.com/cst8207/12f/notes/510_umask.txt) - plain text version of this page in [Pandoc Markdown](http://johnmacfarlane.net/pandoc/) format

## References

* [原文 Permissions masking with umask, chmod, 777 octal permissions](http://teaching.idallen.com/cst8207/12f/notes/510_umask.html)

