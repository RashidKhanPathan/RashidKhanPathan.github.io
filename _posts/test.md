---
title: CVE-2022-44939 DLL Hijacking (Windows LPE) in Efs Software Easy Chat Server v3.1
author: rashidkhanpathan
date: 2022-11-25 14:10:00 +0800
categories: [Windows]
tags: [writing]
pin: tru
render_with_liquid: false
---

## 0x01: Details
- Product: Dual DHCP DNS Server  
- Affected version: V7.40  
- Vendor: Achal Dhir (SourceForge.net)  
- Fixed version: No response from vendor  
- Tested version: V7.40  
- CVE Reference: CVE-2022-46470

## 0x02 Problem Description

The folder permissions on the default installation directory `%SYSTEMDRIVE%\DualServer\` allows anyone in the "Authenticated Users" group to modify its contents. To exploit this vulnerability, a local attacker can replace the `DualServer.exe` with a crafted binary with the same name. Afterwards, if the service is restarted, the attacker's code will be executed in the context of system. The service gets automatically started after a reboot.

## 0x03 Impact

By replacing the binary an attacker can execute code, allowing potential privilege escalation to system.

## 0x04 Workaround

It is advised to change the installation directory to `%SYSTEMDRIVE%\Program Files\` or `%SYSTEMDRIVE%\Program Files(x86)\`. By default, these directories allow "Authenticated Users" to only read and execute the directory contents.

## 0x05 Status

The issue was reported to the vendor, but there was no response.
https://sourceforge.net/p/dhcp-dns-server/discussion/451062/thread/8b15943892/
(i couldn't publish much information about Vulnerabiliy on SourceForge because of 90 Day's Disclosure Policy, also we can't directly disclose the Vulnerability on public platform) i wanted to get Vendor's mail address or contact from SourceForge but Vendor remain silent after 90 Day's here is Disclosure

## 0x06 Disclosure Timeline

2022-11-27: Vulnerability discovered  
2022-11-27: Vulnerability reported to vendor via Sourceforge
2022-11-28: Reported to MITRE for CVE
2022-12-30: 90-day disclosure deadline passed  
2022-12-30: Published advisory

## 0x07 References

[https://sourceforge.net/projects/dhcp-dns-server/](https://sourceforge.net/projects/dhcp-dns-server/)

## 0x08 Proof Of Concept:
[PoC](https://drive.google.com/drive/folders/1iuMwP6Z46XriC0AbqD545tQhSaAJBHTV)


