---
title: Pentesting Git source repositories
---

Attack the source, Luke
-----------------------

During a pentest you sometimes may stumble accross public projects hosted on a git web service. The most commonly found in enterprise environments are BitBucket and GitLab.

You can check these URLs to see if they host any public projects:

* BitBucket: ```https://bitbucket.example.com/repos/visibility#public```
* GitLab: ```https://gitlab.example.com/explore```

From there you may want to dump all public projects to see if there are secrets stored in them:

* BitBucket: [BitBucket repo dumping script](https://gist.github.com/gquere/fbf6fffd36565f15f98923ab6174a9c2)
* GitLab: [GitLab repo dumping script](https://gist.github.com/gquere/ec75dfeefe725a87aada0a09d30962b6)


Patterns of secrets to look for
-------------------------------

From experience, these regular expressions have a high chance of finding cleartext secrets:
```
://[a-zA-Z0-9_-]\+:[^/@]\+@     // passwords in URLs
curl .*-u 
wget .*--password
wget .*--http-password
wget .*--ftp-password
wget .*--proxy-password
Authorization.*:.*Basic
Authorization.*:.*Bearer
docker login .*-p 
<password>[^<]\+</password>
mysql .*-p
mysql .*--password
PGPASSWORD                      // environment variable to set the psql password on the commandline
RSYNC_PASSWORD                  // environment variable to set the rsync password on the commandline
BEGIN PRIVATE
[^a-zA-Z0-9]7z[ zr].*-p[^ ]     // password protected 7z files
unzip .*-P                      // password protected zip files
mount .*-o.*password=           // CIFS share mounted with a username/password
jfrog .*--password=
jfrog .*--apiKey=
mongo .*-p 
cqlsh .*-p 
ldapsearch .*-w 
ldapsearch .*-y 
```

Please submit your own favourites :)

Do not use grep
---------------

Grep is really not recommended once you're working several hundreds of megabytes or even gigabytes of data, as you'll oftentimes find yourself re-running the same commands over and over, adding multiple pipes and pagers along the way. This is a cumbersome process for which I've developed a solution: instead of using grep I've been using [ngp](https://github.com/gquere/ngp2) for years now.

This ncurses tool lets you browse search results in a terminal, execute subsearches on the initial results and open results directly in vim. Try it out the next time you're auditing code!

![](./demo.gif)


Files containing secrets
------------------------

These files usually contain secrets:
```
*id_rsa
*.key
*.ppk
*.secure
.pgpass
.davfs2/secrets
```


Git history
-----------

Git is complex and a lot of people don't know how to properly rebase to hide deleted secrets.

You can recover secrets lost in the repo's history with these projects:

* [TruffleHog](https://github.com/dxa4481/truffleHog)
* [Yar](https://github.com/nielsing/yar)


BitBucket vulnerabilities
-------------------------

BitBucket has an interesting feature for use in redteam engagements: any admin can add a public key to a user. You can use this to commit as another user.

BitBucket version is found in the HTML footer.

### CVE-2019-15000: Arbitrary file read

Applicable to:

* version < 5.16.10
* 6.0.0 <= version < 6.0.10
* 6.1.0 <= version < 6.1.8
* 6.2.0 <= version < 6.2.6
* 6.3.0 <= version < 6.3.5
* 6.4.0 <= version < 6.4.3
* 6.5.0 <= version < 6.5.2

[CVE-2019-15000 PoC here.](https://github.com/86zhou/Poc/blob/master/Bitbucket/CVE-2019-15000.py)


Gitlab vulnerabilities
----------------------

Note that you need to be authenticated to read the version. It's located at this endpoint: ```https://gitlab.example.com/api/v4/version```.

### CVE-2020-10977: Arbitrary file read into RCE < 12.9.1

Requires bug creation and move rights.

[CVE-2020-10977 PoC here.](https://hackerone.com/reports/827052)
