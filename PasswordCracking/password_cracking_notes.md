---
title: Basics of password cracking
---

OpenLDAP formats
================

OpenLDAP dumps from ldapsearch may contain a variety of hash algorithms and formats. Some are not crackable "as is" and will require adaptations:

|OpenLDAP output | Hashcat mode | JtR format       |
|----------------|--------------| ---------------- |
|{SHA}           | 101          | Raw-SHA1         |
|{SHA256}        | 1400 (*)     | Raw-SHA256 (*)   |
|{SHA512}        | 1700 (*)     | Raw-SHA512 (*)   |
|{SSHA}          | 111          | Salted-SHA1      |
|{SSHA512}       | 1711         | SSHA512          |
|{CRYPT}         | 1800 (**)    | sha512crypt (**) |
|{PBKDF2-SHA512} | 12100 (***)  |                  |

(*): Raw-SHA256 and Raw-SHA512 are exported in base64 format, but are expected in hex format. For hashcat, that's ```username:hash``` and for JtR ```username:$SHA256$hash```
(**): Simply remove the '{CRYPT}' string. Note that dots '.' are replacing the '+' characters and are expected to stay like so.
(***): PBKDF2 for hashcat requires several modifications: in the base64 content, the dots '.' become '+'. Also replace '{PBKDF2-SHA512}' with 'sha512:'. Also replace '$' with ':'.


Where to get wordlists?
=======================
* [rockyou](): 10 years later and still a great list with usually a high success rate given its limited size
* [hashes.org](https://hashes.org/left.php): plaintexts of recovered passwords from about all the public leaks of the past 10 years
* [Probable wordlists](https://github.com/berzerk0/Probable-Wordlists/)
* [SecLists](https://github.com/danielmiessler/SecLists)
* depending on your distribution some language dictionaries might be preinstalled under ```/usr/share/dict/```


Where to get mutation rules?
============================
Read [this great article](https://notsosecure.com/one-rule-to-rule-them-all/) about crafting a superset of rules aptly named "One Rule To Rule Them All". You probably won't need another.


In what order to crack?
=======================

The goal here is to maximize efficiency: attack the weakest algorithms first with the fastest rules and then move on to gradually more time-consuming methods, ending with full keyspace exhaustion.

Basics
------
username=password

Dictionnary
-----------
```
john --format=<format> target.txt --wordlist=dictionnary.txt
hashcat --username -m <mode> target.txt dictionnary.txt
```

Dictionary mutations
--------------------
```
john --format=<format> target.txt --wordlist=dictionnary.txt --rules
hashcat --username -m <mode> target.txt dictionnary.txt -r OneRuleToRuleThemAll.rule
```

Masks
-----

Defining custom masks can be especially useful if a restrictive password policy is in place. If, for instance, a company mandates that all passwords should be at least 8 characters long with lowercase, uppercase and specials characters then I can guarantee that this simple mask will recover 20% of the passwords:
```
?u?l?l?l?l?l?l?s
```

Where:
```
?a = all
?s = special, printable non alphanumeric
?l = lowercase
?u = uppercase
?d = digits
```

Defining a custom charset:
```
hashcat --username -m <mode> target.txt -a 3 -1 '?u?d?l*#$@_' '?1?1?1?1?1?1?1?1'
```


This is described in more detail [here for hashcat](https://hashcat.net/wiki/doku.php?id=mask_attack) and [here for John]().


JtR specifics
=============
Installation
------------
Use the bleeding jumbo branch of the [github repo](https://github.com/openwall/john).

Modes
-----
Running John without specifying a dict will cause it to run some interesting basic checks, such as username=password.

formats
-------
redmine: dynamic_1501
user:password$salt

Hashcat specifics
=================
Installation
------------
Use the NVidia official driver. May require you to sometimes hold the kernel back when the ABI is changing (as is currently the case between 5.8 and 5.9). Follow the official installation procedure at []().

Hash format
-----------
username:hash
hashcat --username
hashcat -O: optimized run, can make a HUGE difference depending on the hashing algorithm.
