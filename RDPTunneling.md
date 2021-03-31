From Windows
============

Use [SocksOverRdp](https://github.com/nccgroup/SocksOverRDP).

Needed files
------------
Needed binaries are already compiled in a release archive: https://github.com/nccgroup/SocksOverRDP/releases
```
SocksOverRDP-Plugin.dll
SocksOverRDP-Server.exe
```

Usage
-----
Check the project's [README](https://github.com/nccgroup/SocksOverRDP) for complete instructions.

On windows host, register the DLL:
```
regsvr32.exe SocksOverRDP-Plugin.dll
```

Then RDP to the server:
```
mstsc.exe
```

On windows server, run the binary:
```
SocksOverRDP-Server.exe
```

The SOCKS5 server is then available at 127.0.0.1:1080:
```
ncat.exe --proxy 127.0.0.1:1080 --proxy-type socks5 REMOTECOMPUTER 22
```

From Linux
==========

Use xfreerdp (should be installed on your Kali or available from any decent package manager) and [rdp2tcp](https://github.com/V-E-O/rdp2tcp).

Cross-compile the Windows server
--------------------------------
```
apt install mingw-w64
make server-mingw32
```

Compile the Linux client
------------------------
```
make client
```

Needed files
------------
```
./client/rdp2tcp
./server/rdp2tcp.exe
./tools/rdp2tcp.py
```

Usage
-----
On Linux host:
```
xfreerdp /d:DOMAIN /u:USER /p:PASSWORD /v:COMPUTER:PORT /rdp2tcp:/path/to/rdp2tcp
```

Then on Windows host:
```
rdp2tcp.exe
```

Then on Linux host:
```
echo "socks5 127.0.0.1 1080" >> /etc/proxychains.conf
rdp2tcp.py add socks5 127.0.0.1 1080
```

Then use standard commands with proxychains:
```
proxychains nc -z -v REMOTECOMPUTER 22
```
