---
title: About Jenkins tokens
---

Jenkins tokens are an alternate authentication mechanism to traditional passwords that are used to impersonate a user when performing automated operations, such as triggering a remote build.

This introduces the possibility to delegate rights to robots without supplying them with a plaintext password that would inevitably be stored in a configuration file somewhere and could be used to authenticate to another service if the authentication is for instance based on a LDAP.

Legacy tokens, Jenkins <2.129
-----------------------------
Tokens used to be stored encrypted, and a default token was created for every user. This meant that having read-access to the configuration files enabled impersonation of any user.

The encryption mechanism is the same as for all Jenkins secrets, which I've previously covered [here](https://github.com/gquere/pwn_jenkins#decrypt-jenkins-secrets-offline).

As usual in Jenkins configuration files, the ```{}``` syntax indicated an encrypted secret:
```xml
  <jenkins.security.ApiTokenProperty>
    <apiToken>{AQAAABAAAAAww/xHLk0LqZ8RmVr3o5/gdFuLWXz+NiACjrLU4wiqXJ+vMiuSBWShiyJQtLhV/3UgJxGXTyJJHmLGkMJWGkxi2A==}</apiToken>
  </jenkins.security.ApiTokenProperty>
```

If one has access to the ```master.key``` and ```hudson.util.Secret``` files, it is possible to decrypt it:
```bash
jenkins_offline_decrypt.py  -i .
Encrypted secret: AQAAABAAAAAww/xHLk0LqZ8RmVr3o5/gdFuLWXz+NiACjrLU4wiqXJ+vMiuSBWShiyJQtLhV/3UgJxGXTyJJHmLGkMJWGkxi2A==
29543a2a69b45310fc4b957aa0594e58
```

Updated tokens, Jenkins >=2.129
-------------------------------
Since mid-2018 tokens are stored hashed in the user configuration file, under ```./users/username/config.xml```. The relevant code is [on github](https://github.com/jenkinsci/jenkins/blob/master/core/src/main/java/jenkins/security/apitoken/ApiTokenStore.java).

The token itself is generated from the UI and has to be immediately copied since locally only its hash is stored.

This is an example token:
```1147c3b70068c7b5573342ebb6ddc7f6ae```

It consists of two parts:

* ```11```: this is the hash version. It might evolve someday, but for all intents and purposes if you find a 18-bytes hash that starts with ```11``` then there's a good chance that it's a Jenkins token
* ```47c3b70068c7b5573342ebb6ddc7f6ae```: 16 bytes of random data

Only the random data is stored hashed:
```xml
  <jenkins.security.apitoken.ApiTokenStore_-HashedToken>
    <uuid>d987d752-b057-430e-b701-622872e79196</uuid>
    <name>Token created on 2022-08-18T11:50:55.534+02:00</name>
    <creationDate>2022-08-18 09:50:55.535 UTC</creationDate>
    <value>
      <version>11</version>
      <hash>ff550856862dbee0eb2534d03d4cd9b7a71b2ca35a53e1c1876b4eccfc1e627f</hash>
    </value>
  </jenkins.security.apitoken.ApiTokenStore_-HashedToken>
```

By dropping the version and hashing the value we can verify that the stored hash is indeed the SHA256 sum of the supplied random data:
```bash
echo -n 47c3b70068c7b5573342ebb6ddc7f6ae | sha256sum
ff550856862dbee0eb2534d03d4cd9b7a71b2ca35a53e1c1876b4eccfc1e627f  -
```

This also means that there's no point in trying to bruteforce these hashes since it's equivalent to trying to guess a 16-byte random value.

Token limits
------------
As previously explained tokens are great because they ensure that a leaked secret is not reusable on another service.

But on the other hand using tokens does not exempt you from properly minimizing the privileges attributed to each user. They can still be used to authenticate to the backend and perform operations which can *in fine* execute arbitrary code, such as creating a job, configuring a job or running a groovy script from the Jenkins CLI or using the API:
```
rlwrap java jenkins-cli.jar -s https://buildserver/jenkins -auth jenkins:1147c3b70068c7b5573342ebb6ddc7f6ae groovysh
Groovy Shell (2.4.12, JVM: 1.8.0_302)
Type ':help' or ':h' for help.
-------------------------------------------------------------------------------
"hostname".execute().text
===> buildserver
```
