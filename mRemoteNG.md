---
title: mRemoteNG configuration files
---

During a pentest a colleague came accross a bunch of [mRemoteNG](https://mremoteng.org/) configuration files containing passwords on a network share. There are some scripts online that can decrypt these, but the encryption algorithms have changed a few years back so it was a bit confusing at first, this is my attempt at clearing things up.


Decrypt mRemoteNG configuration files
=====================================

Use [this script](https://github.com/gquere/mRemoteNG_password_decrypt) to decrypt old and new formats of mRemoteNG configuration files.


Versions 1.74 and below
-----------------------
Default master password is 'mR3m'.

Key derivation algorithm is 1 round of md5.

Encryption algorithm is AES-CBC.

Passwords are stored encrypted in base64:
```
initialition_vector (16 bytes)
ciphertext
```


Version 1.75 and newer
-----------------------
In 1.75 the default master password is still 'mR3m'. There were discussions to remove the default password in 1.76 onwards but as of fall 2020 it doesn't seem to have yet changed.

Key derivation algorithm is 1000 rounds of PBKDF2-SHA1.

Encryption algorithm is AES-GCM.

Passwords are stored encrypted in base64:
```
salt (16 bytes)
nonce (16 bytes)
ciphertext
tag (16 bytes)
```


mRemoteNG notes
===============

In the configuration file, the ```Protected``` field is an encrypted dummy value ("ThisIsProtected"/"ThisIsNotProtected") that lets the application quickly verify whether a user password has been set. The application tries to decrypt it at launch and if it fails, it knows it has to ask the user for a password. This can be used to bruteforce custom passwords.

In 1.75 and newer there are alternate ciphers to AES-GCM and the KDF iterations can be tweaked by the user. This is all described in the first line of the configuration file confCons.xml:
```xml
<Connections Name="Connexions" Export="False" EncryptionEngine="AES" BlockCipherMode="GCM" KdfIterations="1000" FullFileEncryption="False" Protected="0RlaSZ8kZayRzE3yO2agQWIXUV5EW3ZWDJ3Pm2SV4yKJaZyYWSxrFgjtbM8RcO1ebkkTuRerKXmfdUmM7oVFZ1M/" ConfVersion="2.6">
```

Each time the application is opened, it will save a backup copy of its configuration in the folder. This can be used to defeat password policies by discovering how users rotate their passwords every other month according to company enforced rotations.

The application is .NET and can be injected with a backdoor using my [DotNetInjector](https://github.com/gquere/DotNetInjector) if it's not installed properly. Do not leave portable applications in shared directories!


Takeaways
=========
In your organisation:

* do not use versions below 1.75
* do not use portable versions of mRemoteNG (or of anything for that matter) as this promotes bad hygiene
* do not rely on the default password to encrypt the configuration file
* do not keep configuration files in world-readable locations
