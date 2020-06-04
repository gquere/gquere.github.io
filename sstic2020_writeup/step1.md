---
title: SSTIC 2020 step 1
---

Challenge information
---------------------

* Challenge type: Memory forensics
* Rating: Easy    2-3 hours

Getting the memory dump
-----------------------

To start our investigation, we're given an archive [which you can download here](https://static.sstic.org/challenge2020/dump.tar.gz).
Once extracted, its contents indicate that this first step is a memory forensics challenge:
```
dump
├── dump_memory.sh
├── extracted
│   └── memory
└── LiME
    ├── doc
    │   ├── external_modules.md
    │   └── README.md
    ├── LICENSE
    ├── README.md
    └── src
        ...
```

The [LiME](https://github.com/504ensicslabs/lime) project is a memory extractor for Linux. It's used to create memory snapshots of a running system.

If you're familiar with CTFs you may want to rush into [volatility](https://github.com/volatilityfoundation/volatility), but let's first take a look at the script first (snipped for brevity's sake):
```bash
# Compile LiME kernel module against target kernel
if [ ! -d "${script_dir}/LiME" ]; then
    echo "LiME not found. Cloning it..."
    git clone -n https://github.com/504ensicsLabs/LiME.git "${script_dir}/LiME"
    cd "${script_dir}/LiME"
    git checkout v1.9
fi
cd "${script_dir}/LiME/src"
make clean
make
if [ ! -r lime-$(uname -r).ko ]; then
    echo "Failed to compile LiME kernel module"
    exit 1
fi
echo "LiME kernel module compiled."

# Dump memory
cd "${output_dir}"
echo "Starting memory dump at $(date +%s)"
insmod "${script_dir}/LiME/src/lime-$(uname -r).ko" "path=$PWD/memory format=padded compress=1 timeout=0"
echo "End of memory dump at $(date +%s) : $(du -h memory | cut -f1)"
```

Ok so the script is probably in someone's memory forensics toolkit. It just grabs LiME, compiles it and then runs it off of the USB stick against the currently running system.
There's one thing to note here: ```compress=1```. Thus we'll need to unzip the generated archive to get to the raw memory.

```
 gquere@sandbox  ~/sstic2020/step1/dump/extracted  file memory
memory: zlib compressed data
 gquere@sandbox  ~/sstic2020/step1/dump/extracted  zlib-flate -uncompress < memory > memory_unzipped
```


Getting the kernel version
--------------------------
In order to properly "read" the memory dump, volatility needs a profile that basically tells it what the kernel offsets and addresses are. Contrary to the limited number of Windows kernels, because there as sooooo many Linux kernels (multiple versions, multiple distributions with multiple lineups each with their own patches ...) you'll generally need to generate your own profile since it doesn't exist. This has become a very common step in CTFs to harden the difficulty a bit.

The first step is to find the kernel version for which we want a profile. This is easily identified using grep or [ngp ;)](https://github.com/gquere/ngp2) :
```
grep -o --text -e 'Linux .* x86_64' memory_unzipped
Linux kazbng-backup 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u2 (2019-11-11) x86_64
Linux kazbng-backup 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u2 (2019-11-11) x86_64
Linux kazboneg-matrix-node 4.19.0-8-amd64 #1 SMP Debian 4.19.98-1 (2020-01-26) x86_64
```

Hum two machines are referenced here: ```kazbng-backup``` and ```kazboneg-matrix-node```. Let's count occurrences of the kernel versions strings to make sure which of these two was the one running:
```
grep -c 4.19.0-6 memory_unzipped
4710
grep -c 4.19.0-8 memory_unzipped
1
```


Creating the volatility profile
-------------------------------
Alright, so we need to generate a volatility profile for Debian with kernel 4.19.0-6. "4.19.0" is a kernel version, and "-6" indicates debian's custom patch level. This is why the kernel ecosystem is a mess.

To generate the profile I started with a Debian 10 vanilla virtual machine. Its currently installed kernel was 4.19.0-8 but I tried it anyways and it didn't work. So I installed the deprecated 4.19.0-6 kernel and headers, then selected it in Grub when booting. You just need to [follow the instructions](https://github.com/volatilityfoundation/volatility/wiki/Linux).

The volatility profile is a zip archive that contains two files:

* Systemp.map
* module.dwarf

The zip file should be placed in ```volatility/plugins/overlays/linux```.

You can verify that it's detected by volatility by running:
```
python vol.py --info | grep -i debian
```

Reading the dump
----------------
Volatility can read files that are still cached. Let's get a list of these:
```
python vol.py --file=memory_unzipped --profile=Linuxdebian-4_19_0-6x64 linux_enumerate_files > files
```

I tweaked the output a bit in vim to get the filename first and the the inode, and deleted files that I wasn't going to look at on the first run (/sys, /proc, /run, /usr, /lib ...).

Then I ran a simple script to dump the remaining files (mostly /etc, /home and /tmp):
```bash
#!/bin/bash

OUT_PATH="./extracted/fs"

while read -r line
do
    inode=$(echo $line | cut -d' ' -f2)
    apath=$(echo $line | cut -d' ' -f1)

    python vol.py --file=memory_unzipped --profile=Linuxdebian-4_19_0-6x64 linux_find_file -i $inode -O "$OUT_PATH/$apath"
done < files
```

In the user's home there's a backup script ```backup.sh```:
```bash
#! /bin/bash
# Custom backup script

set +e

# Logging stuff
LOG_FILE="${HOME}/backup.log"
exec &> ${LOG_FILE}

log() {
	echo "$(LC_ALL=C date --utc "+%F %T") $*"
}

log "Starting backup daemon..."

# Check SSH/PGP config
export SSH_AUTH_SOCK="${XDG_RUNTIME_DIR}/gnupg/S.gpg-agent.ssh"
ssh-add -l

while true
do
	log "Starting new backup"
	WORKDIR=$(mktemp -d)
	BACKUPCMD="KHBnX2R1bXAgc3luYXBzZSA7IHRhciBjeiB+L21lZGlhX3N0b3JlKSB8IGd6aXAgLTEgfCBvcGVuc3NsIGVuYyAtZSAtYWVzMjU2IC1pdiA1ZDExNWEyOWQxZTcwZmM3Y2VmODQ0MWUzYzRmY2I1MyAtSyBlZDM1OGFmNmNlODIyNDgwZDM4NGRmMmNiOTc4NTc2Y2Q1MTI5NzM3MzUzOTkzOGVhOTlmMjY4MTNmZmY1MjVmID4gfi9zeW5hcHNlLmJhawo="
	ssh synapse-node "$(echo ${BACKUPCMD} | base64 -d)"
    # (pg_dump synapse ; tar cz ~/media_store) | gzip -1 | openssl enc -e -aes256 -iv 5d115a29d1e70fc7cef8441e3c4fcb53 -K ed358af6ce822480d384df2cb978576cd51297373539938ea99f26813fff525f > ~/synapse.bak

	log "Downloading backup"
	scp synapse-node:synapse.bak ${WORKDIR}/tmp
	split -b 256k -d ${WORKDIR}/tmp ${WORKDIR}/tmp
	rm ${WORKDIR}/tmp

	log "Copying on external drive"
	sudo mount /dev/disk/by-label/external /mnt/external
	backup_date=$(date "+%F-%T")
	for part in $(ls ${WORKDIR}); do
		gpg --encrypt --armor --recipient "Sivi Ha Kerez" -o /mnt/external/backup/${backup_date}.${part}.backup ${WORKDIR}/${part}
	done
	sha256sum ${WORKDIR}/* > ${WORKDIR}/tmp.sha256sum
	cp ${WORKDIR}/tmp.sha256sum /mnt/external/backup/${backup_date}.sha256sum
	sudo umount /mnt/external # Sync on disk

	# Cleanup
	log "Cleanup"
	srm -r ${WORKDIR}

	# Wait 1 day before next backup
	log "Wait"
	sleep 1d
done
```

Ok so this machine runs a command over SSH on another machine which concatenates a ```pg_dump``` of a PostgreSQL database and a folder archive, gzips it then encrypts it. This database is then copied locally using scp, split into chunks and encrypted again into an external drive using GPG and then ... the drive is unmounted. Uh oh, I don't have this drive anywhere in my snapshot. Luckyly for us there's a fail right afterwards: ```srm -r ${WORKDIR}```. The files were never deleted!

After carving these files from the temporary directory, we just need to stitch them back together using cat (did you know this was cat's original purpose?) and decrypt the archive:
```
cat tmp?? > synapse.bak
openssl enc -d -aes256 -iv 5d115a29d1e70fc7cef8441e3c4fcb53 -K ed358af6ce822480d384df2cb978576cd51297373539938ea99f26813fff525f -in synapse.bak -out synapse_dec.bak
```

And in this dump we get our first flag: ```SSTIC{f8693a5a340b67c4ac13abb93e57da55b935577fc76cbe14c223fb5912ccc795}```.

Next part: [step 2](./step2)
