---
title: Decrypting Informatica secrets
---

During a pentest I got access to the database behind an [Informatica](https://www.informatica.com/) cluster. The DB was Oracle, but I think that PSQL also exists.

Passwords are stored in the ```POU_PASSWORD``` field of the ```PO_USERINFO``` table.

I read [on StackOverflow](https://stackoverflow.com/questions/10795693/reset-informatica-admin-password) that it's possible to decrypt or replace these passwords using the ```pmpasswd``` binary. After some limited amount of research, I don't think that's actually possible.

The ```pmpasswd``` binary is used locally to encrypt passwords to use in scripts. It uses static keys (AES128 for versions <=10.4, AES256 for the >= 10.5 versions). Maybe some of these passwords are saved in the DB, but the user passwords are *not* encrypted using this utility.

The passwords stored in the database are encrypted using a key found on the server in a file named ```siteKey```. This key is generated at install and is derived from a password and the Informatica domain name.

*This means that you cannot decrypt passwords contained in the Informatica database if you only have access to the database.*

Some more notes for posterity:

Informatica 10.4
================

This version uses AES128. The key is contained in bytes [4:20] of the ```siteKey``` file. The IV is constant and is the same as in the ```pmpasswd``` utility. 10 mins in GDB and you'll find it.

The ciphertext of the password is stored in base64 format.

Informatica 10.5
================

This is a bit more complex. The ciphertext is actually contained in a bunch of envelopes encoded in base64:
```
...
00 00 00 00                                   // first envelope
00 00 00 10 ... IV1 .............             // IV to use in combination with siteKey
00 00 00 30 ... ENCRYPTED_KEY ...             // encrypted key (used to decrypt second envelope)
00 00 00 20 ... HMAC1 ...........             // don't care about HMAC
...
00 00 00 10 ... IV2 .............             // IV to use in combination with decrypted key from first envelope
00 00 00 30 ... ENCRYPTED_PASS ..             // user's encrypted pass
00 00 00 20 ... HMAC2 ...........
...
```

Using the AES256 key from ```siteKey``` (bytes [4:36]) and ```IV1```, decrypt the ```ENCRYPTED_KEY```. Use this new key and ```IV2``` to decrypt the ```ENCRYPTED_PASS``` field and recover the plaintext password.
