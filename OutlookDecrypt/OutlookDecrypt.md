---
title: Exporting Outlook Private Keys and decrypting S/MIME emails
---

When someone leaves the company it's usually expected that they will pass on all of their prior emails to their successor. Problem is, most of our work is sensitive and thus most messages are encrypted. Giving away all messages is not enough, as their decryption key is needed as well. The encryption Outlook uses is based on S/MIME which relies on public key cryptography.

These keys (and associated certs) are found in the certicate manager ```certmgr.msc```:

![a](./certmgr.PNG)

Not all certificates are relevant here: we're actually only interested in the first one that contains the private key used to decrypt messages sent to us.

Depending on the company policy, users with limited rights are not allowed to export their private keys. As shown below, the private key export option is grayed out:

![a](./export_grayed_out.PNG)

There are nonetheless two options to export the private keys yourself, but both will trigger your AV software so you'll need a bypass for that.


## The Windows cryptography APIs

There are two cryptography APIs in Windows:

* The legacy [CryptoAPI](https://docs.microsoft.com/en-us/windows/win32/seccrypto/cryptography-portal)
* Its replacement, the ["Cryptographi API: Next Generation"](https://docs.microsoft.com/en-us/windows/win32/seccng/cng-portal), generally referred to as CNG

From what I've seen most applications still use the legacy CryptoAPI, making it rather simple to export private keys as we'll see next.


### CryptoAPI
According to Microsoft's documentation for the [CryptGenKey function](https://docs.microsoft.com/en-us/windows/win32/api/wincrypt/nf-wincrypt-cryptgenkey), there exists a ```CRYPT_EXPORTABLE``` flag:
> If this flag is set, then the key can be transferred out of the CSP into a key BLOB by using the CryptExportKey function. Because session keys generally must be exportable, this flag should usually be set when they are created.
> If this flag is not set, then the key is not exportable. For a session key, this means that the key is available only within the current session and only the application that created it will be able to use it. For a public/private key pair, this means that the private key cannot be transported or backed up.

But the CryptoAPI simply is a userspace DLL that handles keys: we have control over its code from the calling process, which means we also have the ability to patch it.

![a](./CryptoAPI.png)


### CNG
From Win2008 and onwards, a key protection mechanism was introduced in CNG to comply with the [Common Criteria](https://en.wikipedia.org/wiki/Common_Criteria), a security certification standard.

Keys are now fully isolated from the application process that uses them, using Keyiso: the CNG Key Isolation service, hosted in LSASS. This means that any tampering with this service will require admin rights.

![a](./CNG.png)


## Method 1: Mimikatz
### CryptoAPI Patching

For keys protected by the legacy CryptoAPI, no admin rights are needed: simply run mimikatz as the user you want to extract private keys from.

According to mimikatz' [documentation](https://github.com/gentilkiwi/mimikatz/wiki/module-~-crypto#capi), the crypto module has a dedicated ```capi``` function to unprotect private keys:
> This patch modifies a CryptoAPI function in the mimikatz process in order to make unexportable keys exportable (no specific rights other than access to the private key is needed)

Let's first try and export certificates without patching the CryptoAPI:
```
crypto::certificates /export
```

Result:

![a](./mimikatz_certs_only.PNG)

OK, we've got all our certs without the keys, just as what would've happened from the certificate manager had we manually exported the certificates.

Now let's patch the CryptoAPI first before exporting the certificates:

![a](./mimikatz_patch.PNG)

Result:

![a](./mimikatz_patched_certs_and_keys.PNG)

Great, we've now got additional .pfx files which are PKCS12-formatted. Let's have a look inside (default export password is "mimikatz"):
```
openssl pkcs12 -in CURRENT_USER_My_0_xxxxxxxx.pfx -nocerts -nodes
Enter Import Password:
Bag Attributes
    localKeyID: 01 00 00 00
    Microsoft CSP Name: Microsoft Enhanced Cryptographic Provider v1.0
    friendlyName: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Key Attributes
    X509v3 Key Usage: 10
-----BEGIN PRIVATE KEY-----
You wouldn't download a private key?
-----END PRIVATE KEY-----
```

How did this work? [Looking at the code](https://github.com/gentilkiwi/mimikatz/blob/master/mimikatz/modules/crypto/kuhl_m_crypto_patch.c) reveals that it's a rather simple in-memory patch that has to do with jumps, where ```JNE``` is overwritten with ```JMP```:
```C
BYTE PATC_WIN5_CPExportKey_EXPORT[]	= {0xeb};
BYTE PATC_W6AL_CPExportKey_EXPORT[]	= {0x90, 0xe9};
```

Seems like it's enough to skip the unexportable check!


### CNG Patching
Extracting privates keys protected by the CNG will require admin privileges for the patching of LSASS (same as previously, overwrite the ```JMP``` of the export check):
```
mimikatz # privilege::debug
Privilege '20' OK

mimikatz # crypto::cng
"KeyIso" service patched

mimikatz # crypto::certificates /export
...
```

If when patching insted of the KeyIso success message, you get this error:
```
ERROR kull_m_patch_genericProcessOrServiceFromBuild ; OpenProcess (0x00000005)
```
Then it probably means that there's AV software protecting LSASS, which you should be able to confirm by running the ```sekurlsa``` set of commands.

If you get this error:
```
ERROR kull_m_patch_genericProcessOrServiceFromBuild ; kull_m_patch (0x00000000)
```
Then it looks like a failure to patch, either because the version used is not up-to-date with regards to the system, or because LSASS has been patched already.


## Method 2: Jailbreak
This one only applies to CryptoAPI but might be simpler than mimikatz if you don't have an AV bypass method handy, as this software will be a lot easier to recompile/obfuscate.
The [Jailbreak](https://github.com/iSECPartners/jailbreak) project is available on GitHub and includes precompiled binaries that sadly trigger most AVs. But the code is easily read and can be slightly adapted to defeat them.

It simply hooks API calls and let's you use the certificate manager normally, behaving like keys were not protected:

![a](./export_jailbreak.png)


## Opening the S/MIME encrypted messages with Thunderbird
To make sure I had extracted the correct private key, I wanted to open the encrypted emails on Linux.

The ```.msg``` extension indicates that the message is in MS-Outlook format, so we'll need to convert it using ```msgconvert``` which is easily found on Debian-based distributions:
```
apt-cache search msgconvert
libemail-outlook-message-perl - module for reading Outlook .msg files
```

We can then convert our Outlook message to the EML format:
```
msgconvert secret.msg
```

I thought it'd be easily done with Thunderbird, so I imported my .pfx key (Preferences >> Certificates >> Manage Certificates >> Your Certificates >> Import):

![a](./thunderbird_preferences.PNG)

Then it's simply a matter of opening the EML message:
```
thunderbird secret.eml
```

And there we are, we can open the encrypted message on Linux:

![a](./thunderbird.PNG)


## Decrypting S/MIME with plain openssl
We start with the Windows private key in PKCS12 format. Let's convert it to a format usable by openssl:
```
openssl pkcs12 -in CURRENT_USER_My_0_xxxxxxxx.pfx -nocerts -out example.key -nodes
Enter Import Password:

cat example.key
Bag Attributes
    localKeyID: 01 00 00 00
    Microsoft CSP Name: Microsoft Enhanced Cryptographic Provider v1.0
    friendlyName: xxxxxxxxx-MAIL-CHIFFREMENT-OUTLOOK-ac850079-6892-48f6-b327-2090ac4f565e
Key Attributes
    X509v3 Key Usage: 10
-----BEGIN PRIVATE KEY-----
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
-----END PRIVATE KEY-----
```

Akin to what we did for Thunderbird, we need to convert the Outlook file to an EML file:
```
msgconvert secret.msg
```

Let's only keep the P7M portion of the EML. There may be an automated way to do it, I just cropped non-relevant info in vim:
```
cat secret.p7m
Content-Type: application/pkcs7-mime; name="smime.p7m"
Content-Disposition: attachment; filename="smime.p7m"
Content-Transfer-Encoding: base64

MIAGCSqGSIb3DQEHA6CAMIACAQAxggLyMIIBdQIBADBdMFYxCzAJBgNVBAYTAkZSMQ8wDQYDVQQK
DAZFTkVESVMxFzAVBgNVBAsMDjAwMDIgNDQ0NjA4NDQyMR0wGwYDVQQDDBRBQyBNRVNTQUdFUklF
....
```


Then we can decrypt the P7M portion of the EML:
```
openssl smime -decrypt -in secret.p7m -inkey example.key
```

This gives us a big base64 blob way larger than the actual message because it still contains a signature, but if we just take a look at the first few decoded chars we're good to go:
```
echo "MIAGCSqGSIb3DQEHAqCAMIACAQExCzAJBgUrDgMCGgUAMIAGCSqGSIb3DQEHAaCAJIAEggLWQ29u
dGVudC1UeXBlOiBtdWx0aXBhcnQvYWx0ZXJuYXRpdmU7DQoJYm91bmRhcnk9Ii0tLS09X05leHRQ
YXJ0XzAwMF8wMDE1XzAxRDY2RkQ2LjUyQjVGN0YwIg0KDQpUaGlzIGlzIGEgbXVsdGlwYXJ0IG1l
c3NhZ2UgaW4gTUlNRSBmb3JtYXQuDQoNCi0tLS0tLT1fTmV4dFBhcnRfMDAwXzAwMTVfMDFENjZG
RDYuNTJCNUY3RjANCkNvbnRlbnQtVHlwZTogdGV4dC9wbGFpbjsNCgljaGFyc2V0PSJpc28tODg1
OS0xIg0KQ29udGVudC1UcmFuc2Zlci1FbmNvZGluZzogOGJpdA0KDQpTdXBlciBzZWNyZXQgbWVz
c2FnZSENCg0KDQoNCg0KDQoNCl" | base64 -d
Content-Type: multipart/alternative;
	boundary="----=_NextPart_000_0015_01D66FD6.52B5F7F0"

This is a multipart message in MIME format.

------=_NextPart_000_0015_01D66FD6.52B5F7F0
Content-Type: text/plain;
	charset="iso-8859-1"
Content-Transfer-Encoding: 8bit

Super secret message!
```
