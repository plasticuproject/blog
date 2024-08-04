---
layout: post
title: "corCTF2024 shcitty-challenge"
categories: Write-Up
date: 2024-08-04
image: /images/shcitty-challenge/cor.jpg
description: Solution to corCTF2024 forensics challenge "shcitty-challenge", and a general solution to decrypt shc compiled binaries.
tags: [corCTF, Write-Up, Python, C]
katex: true
markup: "markdown"
---

![Image](/images/shcitty-challenge/cor.jpg#center)

******

## The Challenge

![Image](/images/shcitty-challenge/challenge.png#center)

First let's take a look at the files that are provided to us. **terminal_output.txt** appears to be the output the challenge description is referencing. It shows some system information, executes a program called **shc**, as well as a program called **file_information**, both of which produce some output.
```sh
fart@fartbox:~$ uname -a
Linux fartbox 6.5.0-28-generic #29~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC Thu Apr  4 14:39:20 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
fart@fartbox:~$ shc -C
shc Version 4.0.3, Generic Shell Script Compiler
shc GNU GPL Version 3 Md Jahidul Hamid <jahidulhamid@yahoo.com>
shc Copying:

    This program is free software; you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation; either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program; if not, write to the Free Software
    @Neurobin, Dhaka, Bangladesh

    Report problems and questions to:http://github.com/neurobin/shc

    Md Jahidul Hamid <jahidulhamid@yahoo.com>

fart@fartbox:~$ shc -U -H -v -o flag -f flag.sh && rm flag.sh.x.c && rm flag.sh
shc shll=sh
shc [-i]=-c
shc [-x]=exec '%s' "$@"
shc [-l]=
shc opts=
shc: cc   flag.sh.x.c -o flag
shc: strip flag.sh.x
shc: chmod ug=rwx,o=rx flag.sh.x
fart@fartbox:~$ ./file_information /bin/sh

Inode Number: 42467530
Device Number: 2050
Device ID: 0
User ID: 0
Group ID: 0
File Size: 125688
Last Modification Time: 1648043363
Last Status Change Time: 1686442713

fart@fartbox:~$
```

We can see this was run on *Ubuntu 22.04, Linux kernel version 6.5.0*. We also see that the version of **shc** is *4.0.3*, and that it is a *"Generic Shell Script Compiler"*. We have the options and arguments supplied to the program, and see that after successful completion it then removes two files. It's not a huge logical jump to assume that **shc** was ran with these options, supplied with a shell script named **flag.sh**, generated a binary file named **flag**, and some C source file named **flag.sh.x.c**.

We see that **file_information** was ran with the supplied argument **/bin/sh**, and we can assume that's the path to the *system's default shell binary*. Finally, we see the output of **file_information**, which appears to be some of the file [stats](https://linux.die.net/man/2/stat) for **/bin/sh**.


Next we will look at **file_information.c**. This appears to be the source code for the last program that was executed in the captured output above.

```c
#include <stdio.h>     // printf
#include <stdlib.h>    // malloc and free
#include <string.h>    // memcpy
#include <sys/stat.h>  // statf


int file_info(char * file)
{
        struct stat statf[1];
        struct stat control[1];

        if (stat(file, statf) < 0)
                return -1;

        /* Turn on stable fields */
        memset(control, 0, sizeof(control));
        control->st_ino = statf->st_ino;
        control->st_dev = statf->st_dev;
        control->st_rdev = statf->st_rdev;
        control->st_uid = statf->st_uid;
        control->st_gid = statf->st_gid;
        control->st_size = statf->st_size;
        control->st_mtime = statf->st_mtime;
        control->st_ctime = statf->st_ctime;

        // Print readable file information
        printf("\nInode Number: %lu\n", control->st_ino);
        printf("Device Number: %lu\n", control->st_dev);
        printf("Device ID: %lu\n", control->st_rdev);
        printf("User ID: %u\n", control->st_uid);
        printf("Group ID: %u\n", control->st_gid);
        printf("File Size: %ld\n", control->st_size);
        printf("Last Modification Time: %ld\n", control->st_mtime);
        printf("Last Status Change Time: %ld\n\n", control->st_ctime);

        return 0;
}


int main(int argc, char ** argv)
{
        file_info(argv[1]);
        return 1;
}
```

As mentioned above, this does indeed print file stat information for the supplied file.

We can run the **file** command on the **shc** generated **flag** binary to get some information about it. We see it is a *64 bit ELF binary*. No surprises there. When we attempt to execute it as a *non-priviledged user*, we get an *"Operation not permitted"* error. When we try to execute it with *root privileges*, we get garbage.

![Image](/images/shcitty-challenge/file.png#center)

******

## Research

So what the heck is a *"Generic Shell Script Compiler"*, and more specifically **shc** anyway? A quick internet search points us to it's [open source repository on Github](https://github.com/neurobin/shc/tree/4.0.3). The **README.md** states:

> *A generic shell script compiler. Shc takes a script, which is specified on the command line and produces C source code. The generated source code is then compiled and linked to produce a stripped binary executable.*
>
> *The compiled binary will still be dependent on the shell specified in the first line of the shell code (i.e shebang) (i.e. #!/bin/sh), thus shc does not create completely independent binaries.*
>
> *shc itself is not a compiler such as cc, it rather encodes and encrypts a shell script and generates C source code with the added expiration capability. It then uses the system compiler to compile a stripped binary which behaves exactly like the original script. Upon execution, the compiled binary will decrypt and execute the code with the shell -c option.*
>

Simple enough to understand. Let's install the relevant version of **shc**, and run a **man** command to get a description of the options that can be supplied or omitted to the program.

![Image](/images/shcitty-challenge/options.png#center)

The execution in question was run with the arguments *-U* and *-H*, making the generated binary *"untraceable"* and *"hardened"*. It was also **NOT** run with option *-r*, which relaxes security. This prevents execution of the binary on other systems. We'll need to look into how that works in more detail later.

For now, let's create a test script and run **shc** on it using the same arguments as the challenge.

![Image](/images/shcitty-challenge/shc-test.png#center)

Carefully reading the generated **test.sh.x.c** intermediate source file we learn it contains the following:

> -   _**Compilation Information**: The initial comment block indicates the tool (`shc`) used to compile the script and the flags passed to it._
>     
> -   _**Static Data**: The `data` array contains encrypted data used in the obfuscated script._
>     
> -   _**Hardened Execution**: The program includes several hardening techniques:_
>     
>     -   _**ARC4 Encryption/Decryption**: Functions for ARC4 encryption and decryption (`arc4`, `stte_0`, `key`) are used to encrypt/decrypt the script._
>     -   _**Seccomp Sandboxing**: `seccomp_hardening()` sets up a restricted environment to limit the system calls the script can make._
>     -   _**Anti-Debugging**: `untraceable` function prevents tracing and debugging of the script._
>     -   _**Environment Checking**: `chkenv` and `key_with_file` functions check the environment to ensure the script runs in the expected context._
> -   _**Execution**: The `xsh` function handles the actual execution of the obfuscated script:_
>     
>     -   *Decrypts the script and checks its validity.*
>     -   *Sets up arguments for execution.*
>     -   *Executes the decrypted script using `execvp`.*
> -   _**Main Function**: The `main` function initializes hardening mechanisms and invokes the obfuscated script._
> 
&nbsp;

It is designed to securely execute the shell script while protecting its contents from being easily accessed or tampered with. The obfuscation and encryption techniques, along with the hardened execution environment, make it difficult to reverse-engineer the script, but not impossible.

It appears to use the **key_with_file** function with *stat* information from the *system's default shell binary* for the encryption process if the *-r* option is not present. Since we probably do not need to actually execute this binary, but rather just decrypt it to get the original shell script, we can remove a lot of the logic from this intermediate source file while adding print debugging for the decrypted static data and such. Here is our resulting code file:

```c
#include <stdio.h>     // printf
#include <stdlib.h>    // malloc and free
#include <string.h>    // memcpy
#include <sys/stat.h>  // statf

void print_data(const unsigned char *data, int size)
{
        unsigned char *data_copy = (unsigned char *)malloc(size * sizeof(unsigned char));
        if (data_copy == NULL) {
                printf("Memory allocation failed\n");
                return;
        }
        memcpy(data_copy, data, size);
        for (int i = 0; i < size; i++) {
                printf("%c", data_copy[i]);
        }
        printf("\n");
        free(data_copy);
}

static  char data [] = 
#define      lsto_z	1
#define      lsto	((&data[0]))
        "\310"
#define      tst1_z	22
#define      tst1	((&data[4]))
        "\172\256\177\263\325\161\300\077\102\350\050\315\143\205\213\111"
        "\222\276\263\157\210\203\272\164\067\145\234"
#define      date_z	1
#define      date	((&data[28]))
        "\026"
#define      chk1_z	22
#define      chk1	((&data[30]))
        "\307\345\036\003\302\305\135\007\277\302\346\171\226\353\152\235"
        "\230\052\363\321\370\045\314\357\033\052"
#define      rlax_z	1
#define      rlax	((&data[55]))
        "\073"
#define      tst2_z	19
#define      tst2	((&data[59]))
        "\054\152\024\142\234\352\301\342\336\201\050\374\165\054\133\066"
        "\041\174\000\264\341\063"
#define      pswd_z	256
#define      pswd	((&data[133]))
        "\157\127\255\321\321\133\120\066\370\200\045\021\352\130\256\262"
        "\110\311\334\377\250\211\344\114\233\021\267\260\026\221\324\205"
        "\350\201\127\272\334\250\360\325\051\026\346\023\156\224\305\266"
        "\136\241\266\006\052\233\123\377\357\301\175\307\233\157\055\067"
        "\124\057\160\334\213\104\204\333\126\356\150\126\144\215\316\173"
        "\320\132\143\240\141\364\237\121\265\035\031\121\214\106\211\340"
        "\165\372\274\001\076\101\335\224\057\105\353\224\323\272\017\243"
        "\024\163\104\166\150\344\310\036\001\341\160\216\050\371\156\236"
        "\364\053\237\062\155\175\307\235\302\263\062\226\155\101\072\202"
        "\265\177\371\035\143\301\074\145\243\254\363\313\246\142\152\232"
        "\216\012\315\373\207\225\231\112\110\313\341\265\015\033\067\302"
        "\232\061\340\376\362\034\143\226\311\127\142\157\271\314\012\107"
        "\327\327\103\136\154\334\251\265\250\212\152\265\246\242\167\101"
        "\323\130\077\306\164\243\135\075\372\277\255\264\214\267\374\143"
        "\217\077\302\374\034\153\261\305\366\033\172\234\276\362\335\222"
        "\112\035\131\276\300\266\374\273\165\252\157\001\142\154\144\361"
        "\254\046\355\310\222\236\215\210\272\010\045\170\372\003\013\104"
        "\040\144\003\341\032\377\235\220\252\015\222\014\171\367\375\045"
        "\035\353\356\260\211\174\071\103\204\136\274\176\142\310\302\203"
        "\054\306\144\107\305\002\330\306\254\012\166\303\234\112\111\205"
#define      opts_z	1
#define      opts	((&data[398]))
        "\012"
#define      msg2_z	19
#define      msg2	((&data[400]))
        "\175\141\363\021\370\214\253\352\372\100\005\061\201\051\133\303"
        "\005\260\164\047"
#define      text_z	119
#define      text	((&data[427]))
        "\205\265\370\112\154\127\353\042\116\061\200\307\201\302\170\162"
        "\200\247\252\066\366\126\121\025\214\317\245\017\030\313\371\161"
        "\047\010\202\113\035\265\376\312\201\252\030\135\161\073\164\326"
        "\223\174\000\004\263\043\235\166\133\025\212\110\007\224\363\223"
        "\226\254\130\261\205\022\366\053\340\301\076\173\374\022\026\010"
        "\056\047\300\016\064\100\225\134\307\141\066\327\047\031\230\203"
        "\241\130\305\051\341\072\345\117\003\135\235\165\211\334\163\303"
        "\003\376\166\375\014\351\325\026\067\252\037\256\246\225\247\136"
        "\026\276\261\335\153\274\123\056\131\236"
#define      chk2_z	19
#define      chk2	((&data[561]))
        "\027\037\023\140\253\120\000\130\063\277\271\310\122\215\161\151"
        "\301\015\315\107\046\315\071\120\220"
#define      msg1_z	65
#define      msg1	((&data[592]))
        "\127\115\355\241\271\104\215\334\243\244\140\231\343\141\252\136"
        "\314\213\142\242\054\370\055\052\020\275\031\140\012\141\072\057"
        "\146\266\321\065\112\233\173\261\152\256\250\100\021\354\036\303"
        "\370\370\175\017\373\177\154\367\377\376\076\371\307\214\322\341"
        "\160\235\105\057\327\253\001\062\122\311\000\233\124\201\006\021"
        "\325\064\153\164\253\112\336\303\152\361\043"
#define      shll_z	8
#define      shll	((&data[674]))
        "\123\123\263\004\315\245\156\314\327\167\115"
#define      inlo_z	3
#define      inlo	((&data[684]))
        "\216\040\236"
#define      xecc_z	15
#define      xecc	((&data[689]))
        "\067\115\067\310\130\263\264\370\344\271\207\014\273\232\260\222"
        "\041\040"/* End of data[] */;
#define      hide_z	4096

/* 'Alleged RC4' */

static unsigned char stte[256], indx, jndx, kndx;

/*
 * Reset arc4 stte. 
 */
void stte_0(void)
{
        indx = jndx = kndx = 0;
        do {
                stte[indx] = indx;
        } while (++indx);
}

/*
 * Set key. Can be used more than once. 
 */
void key(void * str, int len)
{
        unsigned char tmp, * ptr = (unsigned char *)str;
        while (len > 0) {
                do {
                        tmp = stte[indx];
                        kndx += tmp;
                        kndx += ptr[(int)indx % len];
                        stte[indx] = stte[kndx];
                        stte[kndx] = tmp;
                } while (++indx);
                ptr += 256;
                len -= 256;
        }
}

/*
 * Crypt data. 
 */
void arc4(void * str, int len)
{
        unsigned char tmp, * ptr = (unsigned char *)str;
        while (len > 0) {
                indx++;
                tmp = stte[indx];
                jndx += tmp;
                stte[indx] = stte[jndx];
                stte[jndx] = tmp;
                tmp += stte[indx];
                *ptr ^= stte[tmp];
                ptr++;
                len--;
        }
}

/* End of ARC4 */


int key_with_file(char * file)
{
        struct stat statf[1];
        struct stat control[1];

        if (stat(file, statf) < 0)
                return -1;

        /* Turn on stable fields */
        memset(control, 0, sizeof(control));
        control->st_ino = statf->st_ino;
        control->st_dev = statf->st_dev;
        control->st_rdev = statf->st_rdev;
        control->st_uid = statf->st_uid;
        control->st_gid = statf->st_gid;
        control->st_size = statf->st_size;
        control->st_mtime = statf->st_mtime;
        control->st_ctime = statf->st_ctime;

        // Print readable file information
        printf("\nInode Number: %lu\n", control->st_ino);
        printf("Device Number: %lu\n", control->st_dev);
        printf("Device ID: %lu\n", control->st_rdev);
        printf("User ID: %u\n", control->st_uid);
        printf("Group ID: %u\n", control->st_gid);
        printf("File Size: %ld\n", control->st_size);
        printf("Last Modification Time: %ld\n", control->st_mtime);
        printf("Last Status Change Time: %ld\n\n", control->st_ctime);

        // Print raw bytes of control
        unsigned char *bytes = (unsigned char *)control;
        printf("Control bytes: ");
        for (size_t i = 0; i < sizeof(control); i++) {
                printf("%02x", bytes[i]);
        }
        printf("\n");
        key(control, sizeof(control));
        return 0;
}


char * xsh(int argc, char ** argv)
{
        int rFlag = 0;

        stte_0();
        key(pswd, pswd_z);
        arc4(msg1, msg1_z);
        printf("msg1: ");
        print_data(msg1, msg1_z);
        arc4(date, date_z);
        printf("date: ");
        print_data(date, date_z);
        arc4(shll, shll_z);
        printf("shll: ");
        print_data(shll, shll_z);
        arc4(inlo, inlo_z);
        printf("inlo: ");
        print_data(inlo, inlo_z);
        arc4(xecc, xecc_z);
        printf("xecc: ");
        print_data(xecc, xecc_z);
        arc4(lsto, lsto_z);
        printf("lsto: ");
        print_data(lsto, lsto_z);
        arc4(tst1, tst1_z);
        printf("tst1: ");
        print_data(tst1, tst1_z);
        key(tst1, tst1_z);
        arc4(chk1, chk1_z);
        printf("chk1: ");
        print_data(chk1, chk1_z);
        arc4(msg2, msg2_z);
        printf("msg2: ");
        print_data(msg2, msg2_z);
        arc4(rlax, rlax_z);
        if (!rFlag) {
                key_with_file(shll);
	}
	arc4(opts, opts_z);
        printf("opts: ");
        print_data(opts, opts_z);
	arc4(text, text_z);
        printf("text: ");
        print_data(text, text_z);
	return 0;
}

int main(int argc, char ** argv)
{
        argv[1] = xsh(argc, argv);
        return 1;
}
```

And when we compile and execute, we get:

![Image](/images/shcitty-challenge/test-decrypt.png#center)

It's possible that if we had the relevant *system default shell binary file stats* and the *options* used when running **shc**, we could reverse everything from the binary's **.data** section and obtain the original shell script. As it turns out, we indeed have all of those.

******

## Decrypting to Solve

With our test run, we know what *static data variables* the given **shc** *options* will be decrypted to. With the information given to us by **file_information** in the **terminal_output.txt** file, we know what bytes will be used in the **key_with_file** function. What we don't know are the offset locations of the variables in the **.data** section. A general approach that should work on _any_ **shc** binary generated with these known values, is to just do a brute force search for these offsets during the decryption process, using the decrypted variables as *cribs* and using their lengths as window parameters for the brute search. The final **text** variable will contain the original shell script text. For this we can just search for any *crib* that would be found in the original script. A good guess would be the string *"#!/bin/sh"*.

In our Python solve script we will:

> - _Recreate the **ARC4** algorithm._
> - _Build the **/bin/sh** file stats bytes._
> - _Import the binary's **.data** section._
> - _Set our variable cribs and lengths._
> - _Brute force decrypt sections of the **.data** section in-place, following the prescribed order. Reset the encryption key and restart the decryption process  using the newly found variable location when a crib is found, continuing this process until the algorithm is finished and the original shell script (**text** variable) is decrypted._


******

## Full Solution

```python
"""
With the given shc version and command arguments, you can compile/encrypt a few test scripts and spot some patterns.
Looking at the generated intermediate C files, take note of the ARC4 algo, xsh crypt function, and write a crypter to
decrypt and print the known offsets and lengths of the data array to find byte strings to use for cribbing.
In our solver, load the .data section, clean it, and sliding window brute force each sequential crypt call data section
with our cribs to decrypt the text/original shell script.

If the "-r" was used during shc generation, run the script as-is.

If the "-r" option was NOT used, details about the system default shell binary are used in the encryption process.
You will need to compile and run the below C program ON THE MACHINE USED TO GENERATE THE SHC ENCRYPTED BINARY
to get this information to use in decryption. Save the printed "Control Bytes" output of this program to the
variable "control" in this script, set "RFLAG = False", and run as normal.

\`\`\`c
#include <stdio.h>     // printf
#include <string.h>    // memset
#include <sys/stat.h>  // statf


int key_with_file(char * file)
{
        struct stat statf[1];
        struct stat control[1];

        if (stat(file, statf) < 0)
                return -1;

        /* Turn on stable fields */
        memset(control, 0, sizeof(control));
        control->st_ino = statf->st_ino;
        control->st_dev = statf->st_dev;
        control->st_rdev = statf->st_rdev;
        control->st_uid = statf->st_uid;
        control->st_gid = statf->st_gid;
        control->st_size = statf->st_size;
        control->st_mtime = statf->st_mtime;
        control->st_ctime = statf->st_ctime;

        // Print readable file information
        printf("\nInode Number: %lu\n", control->st_ino);
        printf("Device Number: %lu\n", control->st_dev);
        printf("Device ID: %lu\n", control->st_rdev);
        printf("User ID: %u\n", control->st_uid);
        printf("Group ID: %u\n", control->st_gid);
        printf("File Size: %ld\n", control->st_size);
        printf("Last Modification Time: %ld\n", control->st_mtime);
        printf("Last Status Change Time: %ld\n\n", control->st_ctime);

        // Print raw bytes of control
        unsigned char *bytes = (unsigned char *)control;
        printf("\nControl bytes: ");
        for (size_t i = 0; i < sizeof(control); i++) {
                printf("%02x", bytes[i]);
        }
        printf("\n\n");
        return 0;
}

int main()
{
        char *filepath = "/bin/sh";
        int result = key_with_file(filepath);
        if (result == -1) {
                printf("Failed to retrieve file stats.\n\n");
        }
        return 0;
}
\`\`\`

In our "teminal_output.txt" we can see that a similar program is run by the user, giving us the
the "control bytes" information we need for the host default shell binary, so in the "file_information.c"
file, we can just fill in those values and print out the "control" struct hex bytes as shown above, or
use the "get_control_bytes" function below.
"""
import struct
import lief


def get_control_bytes(st_dev, st_ino, st_id, st_gid, st_rdev, st_size,
                      st_mtime, st_ctime):
    """Generate a byte sequence representing a structured control block."""

    # Define the values
    control_bytes = struct.pack(

        # Types are not correct, I may have even messed 
        #  the order up but I don't care because it works
        "QQQQIIQQQQQQQQQQQQQ",  # Format string

        st_dev,  # 8 bytes
        st_ino,  # 8 bytes
        0,  # 8 bytes Unused
        0,  # 8 bytes Unused
        st_id,  # 4 bytes
        st_gid,  # 4 bytes
        st_rdev,  # 8 bytes
        st_size,  # 8 bytes
        0,  # 8 bytes Unused
        0,  # 8 bytes Unused
        0,  # 8 bytes Unused
        0,  # 8 bytes Unused
        st_mtime,  # 8 bytes
        0,  # 8 bytes Unused
        st_ctime,  # 8 bytes
        0,  # 8 bytes Unused
        0,  # 8 bytes Unused
        0,  # 8 bytes Unused
        0,  # 8 bytes Unused
    )
    return control_bytes


# 'Alleged RC4' #

# Global state for the RC4 cipher
stte = list(range(256))
indx = 0
jndx = 0
kndx = 0


def stte_0():
    """Reset the RC4 state."""
    global stte, indx, jndx, kndx
    indx = jndx = kndx = 0
    stte = list(range(256))


def key(data):
    """Set the key for RC4. This can be called multiple times."""
    global stte, indx, kndx
    len_data = len(data)
    while len_data > 0:
        while indx < 256:
            tmp = stte[indx]
            kndx = (kndx + tmp + data[indx % len(data)]) % 256
            stte[indx], stte[kndx] = stte[kndx], stte[indx]
            indx += 1
        data = data[256:]  # Move the pointer by 256 bytes in the array
        len_data -= 256
        indx = 0  # Reset indx for next block


def arc4(data):
    """Encrypt or decrypt data in-place using the current RC4 state."""
    global stte, indx, jndx
    n = len(data)
    output = bytearray(data)
    for i in range(n):
        indx = (indx + 1) % 256
        jndx = (jndx + stte[indx]) % 256
        stte[indx], stte[jndx] = stte[jndx], stte[indx]
        t = (stte[indx] + stte[jndx]) % 256
        output[i] ^= stte[t]
    data[:] = output  # Update the original data in-place


# End of ARC4 #


# Binary file to crack
FILE = "flag"

# Crib text to use for the original shell script
CRIB = "#!/bin/sh"

# Set False if shc was run without the -r option 
RFLAG = False

# File stat details of the host system default shell binary used for encryption
DEVICE_NUMBER = 2050
INODE_NUMER = 42467530
USER_ID = 0
GROUP_ID = 0
DEVICE_ID = 0
FILE_SIZE = 125688
LAST_MODIFICATION_TIME = 1648043363
LAST_STATUS_CHANGE_TIME = 1686442713

try:
    control = get_control_bytes(DEVICE_NUMBER, INODE_NUMER, USER_ID, GROUP_ID,
                                DEVICE_ID, FILE_SIZE, LAST_MODIFICATION_TIME,
                                LAST_STATUS_CHANGE_TIME)
except:
    control = b'\x00'

# Load the ELF file
elf_file = lief.parse(FILE)

# Find the .data section
data_section = elf_file.get_section(".data")
if data_section:
    # Get the content of the .data section as a list of bytes (integers)
    # Convert list of integers to a bytearray
    # Remove the first 32 bytes
    # Strip trailing null bytes (0x00)
    data = bytearray(data_section.content)[32:].rstrip(b'\x00')

# Data and pswd size
data_size = len(data)
pswd_size = 256

# # Cribs; may need to adjust depending on options used when shc was run

# Round 1 cribs
msg1_content = b'has expired!\nPlease contact your provider jahidulhamid@yahoo.com\x00'
msg1_size = len(msg1_content)
date_content = b'\x00'
date_size = len(date_content)
shll_content = b'/bin/sh\x00'
shll_size = len(shll_content)
inlo_content = b'-c\x00'
inlo_size = len(inlo_content)
xecc_content = b'exec \'%s\' "$@"\x00'
xecc_size = len(xecc_content)
lsto_content = b'\x00'
lsto_size = len(lsto_content)
tst1_content = b'location has changed!\x00'
tst1_size = len(tst1_content)

# Round 2 cribs
chk1_content = b'location has changed!\x00'
chk1_size = len(chk1_content)
msg2_content = b'abnormal behavior!\x00'
msg2_size = len(msg2_content)
rlax_content = b'\x92'
rlax_size = len(rlax_content)
opts_content = b'\x00'
opts_size = len(opts_content)
chk2_size = 19 # Not used
tst2_size = 19 # Not used

# Largest possible text size
p_text_size = data_size - sum([
    pswd_size, msg1_size, date_size, shll_size, inlo_size, xecc_size,
    lsto_size, tst1_size, chk1_size, msg2_size, rlax_size, opts_size,
    chk2_size, tst2_size
])

# Get key and round 1 crypt
for pswd_s_idx in range(data_size - pswd_size + 1):
    pswd = data[pswd_s_idx:pswd_s_idx + pswd_size]
    for msg1_s_idx in range(data_size - msg1_size + 1):
        msg1 = data[msg1_s_idx:msg1_s_idx + msg1_size]
        stte_0()
        key(pswd)
        arc4(msg1)
        if msg1 == msg1_content:
            # print(msg1)

            for date_s_idx in range(data_size - date_size + 1):
                date = data[date_s_idx:date_s_idx + date_size]
                stte_0()
                key(pswd)
                arc4(msg1)
                arc4(date)
                if date == date_content:
                    # print(date)

                    for shll_s_idx in range(data_size - shll_size + 1):
                        shll = data[shll_s_idx:shll_s_idx + shll_size]
                        stte_0()
                        key(pswd)
                        arc4(msg1)
                        arc4(date)
                        arc4(shll)
                        if shll == shll_content:
                            # print(shll)

                            for inlo_s_idx in range(data_size - inlo_size + 1):
                                inlo = data[inlo_s_idx:inlo_s_idx + inlo_size]
                                stte_0()
                                key(pswd)
                                arc4(msg1)
                                arc4(date)
                                arc4(shll)
                                arc4(inlo)
                                if inlo == inlo_content:
                                    # print(inlo)

                                    for xecc_s_idx in range(data_size - xecc_size + 1):
                                        xecc = data[xecc_s_idx:xecc_s_idx + xecc_size]
                                        stte_0()
                                        key(pswd)
                                        arc4(msg1)
                                        arc4(date)
                                        arc4(shll)
                                        arc4(inlo)
                                        arc4(xecc)
                                        if xecc == xecc_content:
                                            # print(xecc)

                                            for lsto_s_idx in range(data_size - lsto_size + 1):
                                                lsto = data[lsto_s_idx:lsto_s_idx + lsto_size]
                                                stte_0()
                                                key(pswd)
                                                arc4(msg1)
                                                arc4(date)
                                                arc4(shll)
                                                arc4(inlo)
                                                arc4(xecc)
                                                arc4(lsto)
                                                if lsto == lsto_content:
                                                    # print(lsto)

                                                    for tst1_s_idx in range(data_size - tst1_size + 1):
                                                        tst1 = data[tst1_s_idx:tst1_s_idx + tst1_size]
                                                        stte_0()
                                                        key(pswd)
                                                        arc4(msg1)
                                                        arc4(date)
                                                        arc4(shll)
                                                        arc4(inlo)
                                                        arc4(xecc)
                                                        arc4(lsto)
                                                        arc4(tst1)
                                                        if tst1 == tst1_content:
                                                            # print(tst1)

                                                            # Round 2 crypt, preserve keys
                                                            for chk1_s_idx in range(data_size - chk1_size + 1):
                                                                pswd = data[pswd_s_idx:pswd_s_idx + pswd_size]
                                                                tst1 = data[tst1_s_idx:tst1_s_idx + tst1_size]
                                                                chk1 = data[chk1_s_idx:chk1_s_idx + chk1_size]
                                                                stte_0()
                                                                key(pswd)
                                                                arc4(msg1)
                                                                arc4(date)
                                                                arc4(shll)
                                                                arc4(inlo)
                                                                arc4(xecc)
                                                                arc4(lsto)
                                                                arc4(tst1)
                                                                key(tst1)
                                                                arc4(chk1)
                                                                if chk1 == chk1_content:
                                                                    # print(chk1)

                                                                    for msg2_s_idx in range(data_size - msg2_size + 1):
                                                                        pswd = data[pswd_s_idx:pswd_s_idx + pswd_size]
                                                                        tst1 = data[tst1_s_idx:tst1_s_idx + tst1_size]
                                                                        msg2 = data[msg2_s_idx:msg2_s_idx + msg2_size]
                                                                        stte_0()
                                                                        key(pswd)
                                                                        arc4(msg1)
                                                                        arc4(date)
                                                                        arc4(shll)
                                                                        arc4(inlo)
                                                                        arc4(xecc)
                                                                        arc4(lsto)
                                                                        arc4(tst1)
                                                                        key(tst1)
                                                                        arc4(chk1)
                                                                        arc4(msg2)
                                                                        if msg2 == msg2_content:
                                                                            # print(msg2)

                                                                            for rlax_s_idx in range(data_size - rlax_size + 1):
                                                                                pswd = data[pswd_s_idx:pswd_s_idx + pswd_size]
                                                                                tst1 = data[tst1_s_idx:tst1_s_idx + tst1_size]
                                                                                rlax = data[rlax_s_idx:rlax_s_idx + rlax_size]
                                                                                stte_0()
                                                                                key(pswd)
                                                                                arc4(msg1)
                                                                                arc4(date)
                                                                                arc4(shll)
                                                                                arc4(inlo)
                                                                                arc4(xecc)
                                                                                arc4(lsto)
                                                                                arc4(tst1)
                                                                                key(tst1)
                                                                                arc4(chk1)
                                                                                arc4(msg2)
                                                                                arc4(rlax)

                                                                                for opts_s_idx in range(data_size - opts_size + 1):
                                                                                    pswd = data[pswd_s_idx:pswd_s_idx + pswd_size]
                                                                                    tst1 = data[tst1_s_idx:tst1_s_idx + tst1_size]
                                                                                    opts = data[opts_s_idx:opts_s_idx + opts_size]
                                                                                    stte_0()
                                                                                    key(pswd)
                                                                                    arc4(msg1)
                                                                                    arc4(date)
                                                                                    arc4(shll)
                                                                                    arc4(inlo)
                                                                                    arc4(xecc)
                                                                                    arc4(lsto)
                                                                                    arc4(tst1)
                                                                                    key(tst1)
                                                                                    arc4(chk1)
                                                                                    arc4(msg2)
                                                                                    arc4(rlax)
                                                                                    if not RFLAG:
                                                                                        key(control)
                                                                                    arc4(opts)
                                                                                    if opts == opts_content:
                                                                                        # print(opts)

                                                                                        while p_text_size > 0:
                                                                                            for text_s_idx in range(data_size):
                                                                                                pswd = data[pswd_s_idx:pswd_s_idx + pswd_size]
                                                                                                tst1 = data[tst1_s_idx:tst1_s_idx + tst1_size]
                                                                                                text = data[text_s_idx:text_s_idx + data_size]
                                                                                                stte_0()
                                                                                                key(pswd)
                                                                                                arc4(msg1)
                                                                                                arc4(date)
                                                                                                arc4(shll)
                                                                                                arc4(inlo)
                                                                                                arc4(xecc)
                                                                                                arc4(lsto)
                                                                                                arc4(tst1)
                                                                                                key(tst1)
                                                                                                arc4(chk1)
                                                                                                arc4(msg2)
                                                                                                arc4(rlax)
                                                                                                if not RFLAG:
                                                                                                    key(control)
                                                                                                arc4(opts)
                                                                                                arc4(text)
                                                                                                if CRIB.encode("utf-8") in text:
                                                                                                    print("\n" + text[:p_text_size].decode("utf-8", errors="ignore"))
                                                                                                    # print("\n", text[:p_text_size])
                                                                                                    quit()
```

We run our program and are greeted with the flag.

![Image](/images/shcitty-challenge/solve.png#center)

******
