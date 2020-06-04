---
layout: post
title: "CyberSecLabs Plastic Preservation Write-Up"
categories: Write-Up
date: 2020-05-29
image: /blog/images/shares/pics/CyberSecLabs.png
description: Plastic Preservation is a CyberSecLabs challenge I created where we defeat an encryption function written in python.
tags: [CyberSecLabs, Write-Up, Python, Cryptography]
katex: true
markup: "mmark"
---

![CyberSecLabs](/blog/images/shares/pics/CyberSecLabs.png#center)

---
## The Challenge

After downloading and unpacking the archive, we initially see it contains a text file named **encrypted.txt**, a python script called **password_encrypter.py**, and a password protected zipped directory named **flag.txt.zip**, which contains the text file **flag.txt**. It's probably safe to assume that we need to discover a password to unlock flag.txt.zip and read flag.txt, which will contain the challenge flag.

Printing encrypted.txt reveals that it contains what is most likely a **base-64** string.
```python
YzQwZDFmMWI3YmRkNjAxMg==
```
Decoding this reveals what looks like a **hexadecimal** value:
```bash
plasticuproject@UBOX:~$ echo "YzQwZDFmMWI3YmRkNjAxMg==" | base64 -d
c40d1f1b7bdd6012
```
Trying to converting this to **ascii** renders **non-printable characters**, and trying the base-64 string, hexadecimal, and decimal representations will not unlock the flag file. Let's move on and take a look at **password_encrypter.py**:
```python
#!/usr/bin/env python3

import base64
from datetime import datetime


def z(v):
    a = '\x2e\x6c\x6f\x67'
    with open(a, 'a') as a:
        a.write('# {} {} \n'.format(datetime.now(), v))


def y(a):
    b = 0xCBF29CE484222325
    c = '100000001B3'
    for i in a:
        z(str(b))                          # <- #REMOVE#
        b = b ^ ord(str(i)) * int(c, 16)
    b ^= 0xFFFFFFFFFFFFFFF
    d = str.encode(hex(b)[2:])
    return base64.b64encode(d).decode()


def x():
    e = input('[*]Enter your password: ')
    f = y(e)
    print('[*]Your encrypted password is: {}'.format(f))
    print('[*]And has been saved to encrypted.txt')
    with open('encrypted.txt', 'w') as g:
        g.write(f)


if __name__ == '__main__':
    x()
```
The first thing I notice about this script is that someone attempted to obfuscate some of this code. There are objects in here that are written in types to make this file harder to read, but otherwise do nothing else. Let's go ahead and rewrite this, trying to make it easier to read and make more sense:
```python
#!/usr/bin/env python3

import base64
from datetime import datetime


def z(v):
    a = '.log' # changed to UTF-8
    with open(a, 'a') as a:
        a.write('# {} {} \n'.format(datetime.now(), v))


def y(a):
    b = 14695981039346656037 # changed to int type
    c = 1099511628211 # changed to int type
    for i in a:
        z(str(b))                          # <- #REMOVE#
        b = b ^ ord(str(i)) * c # removed now unnecessary type conversion 
    b ^= 1152921504606846975 # changed to int type
    d = str.encode(hex(b)[2:])
    return base64.b64encode(d).decode()


def x():
    e = input('[*]Enter your password: ')
    f = y(e)
    print('[*]Your encrypted password is: {}'.format(f))
    print('[*]And has been saved to encrypted.txt')
    with open('encrypted.txt', 'w') as g:
        g.write(f)


if __name__ == '__main__':
    x()
```
Next let's try to guess what some of these functions do and what these variables are, then give them proper names. We will add some comments to help us understand what this code is doing:
```python
#!/usr/bin/env python3

import base64
from datetime import datetime

# writes input log_value to '.log' file with timestamp
def log_to_file(log_value):
    file_name = '.log'
    with open(file_name, 'a') as log_file:
        log_file.write('# {} {} \n'.format(datetime.now(), log_value))


# encrypts string
def encrypt(input_string):

    # offset
    b = 14695981039346656037

    # prime value
    c = 1099511628211

    # for each character in input_string
    for i in input_string:

        # writes new b value to '.log' file
        log_to_file(str(b))                          # <- #REMOVE#

        # performs XOR on b and string value, MULTIPLIES with prime value and saves to new b
        b = b ^ ord(str(i)) * c

    # XOR b with this value after final character logic
    b ^= 1152921504606846975

    # convert result to hexadecimal 
    result = str.encode(hex(b)[2:])

    # return base64 encoded hexadecimal result
    return base64.b64encode(result).decode()


# main function
def main():

    # enter plaintext password
    password = input('[*]Enter your password: ')

    # encrypts password
    encrypted_password = encrypt(password)

    # prints encrypted password
    print('[*]Your encrypted password is: {}'.format(encrypted_password))
    print('[*]And has been saved to encrypted.txt')

    # writes encrypted password to 'encrypted.txt' file
    with open('encrypted.txt', 'w') as encrypted_password_file:
        encrypted_password_file.write(encrypted_password)


# runs main function when file is called
if __name__ == '__main__':
    main()
```
Now it is much easier to understand what this script it doing! It appears to take a plaintext password, encrypts it with some logic, and returns a sort of base-64 encrypted value. One thing I noticed is what appears to be some debug logging code, where it writes to **.log** at each step of the function where it processes a new character, before the final XOR. We did not see this file the first time we looked because it is a **dotfile** and is hidden. Let's read the **.log** file and see what is inside of it:
```
# DEBUG VALUES
# 2020-05-21 18:03:15.119201 14695981039346656037 
# 2020-05-21 18:03:15.119397 14696061303695535806 
# 2020-05-21 18:03:15.119572 14696043711509479214 
# 2020-05-21 18:03:15.119720 14695985437393189345 
# 2020-05-21 18:03:15.119860 14696089890997852300 
# 2020-05-21 18:03:15.119998 14695972243253652829 
# 2020-05-21 18:03:15.120137 14696084393439713207 
# 2020-05-21 18:03:15.120273 14696031616881559079 
# 2020-05-21 18:03:15.120410 14696017323230383122 
# 2020-05-21 18:03:15.120544 14696058005160652159 
# 2020-05-21 18:03:15.120679 14695979939835027684 
# 2020-05-21 18:03:15.120813 14695997532021092724 
# 2020-05-21 18:03:15.120947 14696053607114127291 
# 2020-05-21 18:03:15.121079 14695998631532721677 
# 2020-05-21 18:03:15.121213 14696076696858318688 
# 2020-05-21 18:03:15.121347 14695953551555979568 
# 2020-05-21 18:03:15.121478 14696084393439700139 
# 2020-05-21 18:03:15.121608 14696034915416472030 
# 2020-05-21 18:03:15.121737 14695990934951315814 
# 2020-05-21 18:03:15.121864 14695973342765258998 
# 2020-05-21 18:03:15.121995 14696085492951327260 
# 2020-05-21 18:03:15.122126 14695989835439670129 
# 2020-05-21 18:03:15.122255 14696041512486226244 
# 2020-05-21 18:03:15.122389 14696055806137376749 
# 2020-05-21 18:03:15.122520 14695963447160612969
```
We can see that when **password_encrypter.py** was last run, it wrote that **b** value to this file at every step of the encryption. Since there were no destructive operations in the function, and we have a newly created value at every step of the encryption saved, all information was preserved and we should be able to rewrite the function, feeding it the encrypted base-64 output as it's input and reversing the encryption logic using our debug values.

---

## Solve Script

The first thing we will do is read those debug values from the **.log** file and place them in a list. We do that by writing code to read each line of the file, splitting that line into a list of four elements using a space as a delimiter, and appending the fourth element, the debug value, to a list. We will also import the **base64** module, as we will need it later. The code will look like this:
```python
import base64


# list to hold our debug values
debugs = []

# open .log file
with open('.log', 'r') as log_file:

    # get each line from file
    lines = log_file.read().split('\n')

# from each line except the first and last, which contain
# "# DEBUG VALUES" and an empty line, respectively
for line in lines[1:-1]:

    # place the fourth item, the debug value, into debug list
    debugs.append(int(line.split()[3]))
```
Now we will write our function to reverse the encryption. We will do this by feeding it our base-64 encrypted password, converting it back to an integer, and performing the encryption function logic to it in reverse using the values in our **debug** list. When we calculate each character we will save that to a list, building the original password backwards. When we are done we will reverse the order of this list, and we should have the original password that was fed into **password_encrypter.py** that produced the base-64 string in **encrypted.txt**. We will print that password and quit the program. Our decryption function looks as follows:
```python
# decryption function
def decrypt(passwd, debugs):

    # where we store characters
    solve = []

    # offset
    b = 14695981039346656037

    # prime value
    c = 1099511628211

    # convert password back to integer
    passwd = int(base64.b64decode(str.encode(passwd)).decode(), 16)

    # final XOR becomes our first
    passwd ^= 1152921504606846975

    # append passwd integer to debug list
    debugs.append(passwd)

    # reverse debug list
    debugs = debugs[::-1]

    # append seed value to debug list
    debugs.append(b)

    # loop through debug list and perform reverse encryption logic
    # and append characters to character list
    for i in range(len(debugs) - 1):
        solve.append(chr(int((debugs[i] ^ debugs[i+1]) / c)))

    # put password in correct order, print and quit
    print( 'PASSWORD: ' + ''.join(solve[::-1]))
    quit()
```
Now at the bottom we will call the function with the base-64 string and debug values as arguments:
```python
decrypt('YzQwZDFmMWI3YmRkNjAxMg==', debugs)
```
This is what our full script will look like:
```python
import base64


# list to hold our debug values
debugs = []
with open('.log', 'r') as log_file:
    lines = log_file.read().split('\n')
for line in lines[1:-1]:
    debugs.append(int(line.split()[3]))


# decryption function
def decrypt(passwd, debugs):
    solve = []
    b = 14695981039346656037
    c = 1099511628211
    passwd = int(base64.b64decode(str.encode(passwd)).decode(), 16)
    passwd ^= 1152921504606846975
    debugs.append(passwd)
    debugs = debugs[::-1]
    debugs.append(b)
    for i in range(len(debugs) - 1):
        solve.append(chr(int((debugs[i] ^ debugs[i+1]) / c)))
    print( 'PASSWORD: ' + ''.join(solve[::-1]))
    quit()


print(decrypt('YzQwZDFmMWI3YmRkNjAxMg==', debugs))
```
This will give us the following output:
```
PASSWORD: y0u_kn0w_y0ur_py7h0n_w3ll
```
We now have the original password that was fed into **password_encrypter.py** and can use it to unlock **flag.txt.zip**.

![password](/blog/images/plastic_preservation/password.png)

We can now read **flag.txt** and solve the challenge.

![flag](/blog/images/plastic_preservation/flag.png)

I designed this challenge in a way where you would really need to understand what the python code is doing in order to reverse the encryption. It is also a good example of how you can rebuild anything if there is total information preservation.

---

## Supplemental Information

The logic I used in this challenge for the "encryption" is a variation of a [FNV-1a](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function) non-cryptographic hashing function that I learned of while reviewing source code for a popular video game written in C#: <sup id="cite_note-1">[[1]](#cite_ref-1)</sup>
```c
public static ulong Hash(string input)
    {
            ulong num = 14695981039346656037;
            for (int index = 0; index < input.Length; ++index)
                num = (ulong)(((long)num ^ (long)input[index]) * 1099511628211L);
            num &= 0xFFFFFFFFFFFFFFF;
            return num;
    }
```
This function was used in the game code to hash asset names. Looking at this original function you will notice it uses an _AND_ operator as it's final [Bitwise Operation](https://en.wikipedia.org/wiki/Bitwise_operation). This is used for [Bitmasking](https://en.wikipedia.org/wiki/Mask_(computing)#Hash_tables) and controls the length of the output hash. This would have discarded bits and prevented us from being able to reverse the hash. I replaced the _AND_ operation with an _XOR_ operation in the challenge function. This allowed us to reverse the hash due to it's [Associative Property](https://en.wikipedia.org/wiki/Associative_property#Definition). The _AND_ operation is a [Logical Conjunction](https://en.wikipedia.org/wiki/Logical_conjunction) bitwise operation. Looking at the Truth Table for this operation you can see that it is not always possible to reconstruct $$p$$, given $$q$$ and $$p\&q$$. <sup id="cite_note-2">[[2]](#cite_ref-2)</sup>

> _Note: This image uses the "**^**" character in place of an "**&**", but it still represents an **AND** operation._

![truth_table](/blog/images/plastic_preservation/logical_conjunction.png#center)

If we substitute _T_ with _1_, and _F_ with _0_ we get bitwise operations.

![bitwise](/blog/images/plastic_preservation/bitwise.png#center)

As you can see, we cannot simply compute the operation in reverse to reproduce the first value - it does not have an  Associative Property. The _AND_ operation is [destructive](https://stackoverflow.com/questions/2566225/how-to-reverse-bitwise-and-in-c#2566235) - it throws information away. Let's compare this to the Truth Table for the [Exclusive Disjunction](https://en.wikipedia.org/wiki/Exclusive_or) _XOR_ operation that we replaced it with, which has the Associative Property. <sup id="cite_note-3">[[3]](#cite_ref-3)</sup>

![truth_table](/blog/images/plastic_preservation/exclusive_disjunction.png#center)

It is clear that every operation can be computed both forward and backward. In order to make this function reversible for the challenge I had to remove the logical _AND_, and replace it with a non-destructive _XOR_ operation to retain total information preservation, hence the name given to this challenge.

---

#### References:

1. <sup id="cite_ref-1">[\^](#cite_note-1)</sup> Landon, C. N. ["FNV Hash"](http://www.isthe.com/chongo/tech/comp/fnv/index.html#FNV-1a)
2. <sup id="cite_ref-2">[\^](#cite_note-2)</sup> Wikipedia. ["Truth Table; Logical Conjunction"](https://en.wikipedia.org/wiki/Truth_table#Logical_conjunction_(AND))
3. <sup id="cite_ref-3">[\^](#cite_note-3)</sup> Wikipedia. ["Truth Table; Exclusive Disjunction"](https://en.wikipedia.org/wiki/Truth_table#Exclusive_disjunction)

---
