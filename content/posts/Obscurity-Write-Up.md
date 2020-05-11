---
layout: post
title: "HackTheBox Obscurity Write-Up"
categories: Write-Up
date: 2020-05-08
image: /blog/images/obscurity/obscurity.png
description: Obscurity is a medium difficulty box where we will leverage bad server code to inject and run commands, and take advantage of poor cryptography and leftover files to get user access. From there we take advantage of sudo privileges and a poorly executed program to read the root.txt file.
tags: [HTB, Write-Up]
katex: true
markup: "mmark"
---

![](/blog/images/obscurity/obscurity.png#center)

***

## Foothold / www-data

We first start off with a **nmap** scan of the machine IP Address. Then we convert our finished scan to a html file and view it in the browser.

![](/blog/images/obscurity/pics/user/1.png)
![](/blog/images/obscurity/pics/user/6.png)


Here we see there is a **Web Server** running on **port 8080**, and a listening **SSH Server** on **port 22**.

![](/blog/images/obscurity/pics/user/7.png)
![](/blog/images/obscurity/pics/user/8.png)


As always we will add the address to our **/etc/hosts** file.

![](/blog/images/obscurity/pics/user/9.png)


Next we check out the web page being served on **port 8080** in a browser. There isn't much there, but it does tell us that the company makes, uses, and distributes custom **cryptographic software**. We see there is possible **username/contact information**. There's also a message to their developers about the web server **source code** file. It tells the **filename**, the **language** it's written in, and that it is in a **_secret_ directory**.

![](/blog/images/obscurity/pics/user/10.png)
![](/blog/images/obscurity/pics/user/11.png)
![](/blog/images/obscurity/pics/user/12.png)
![](/blog/images/obscurity/pics/user/13.png)


We use **wfuzz** to brute force find this _secret_ directory and locate the server **source code**, which will be aided by the fact that we know the name of the source code file. We are successful and find the file located at **../develop/SuperSecureServer.py**.

![](/blog/images/obscurity/pics/user/14.png)


We download the **source code** file.

![](/blog/images/obscurity/pics/user/15.png)


Then open and inspect the code, looking for vulnerabilities.

![](/blog/images/obscurity/pics/user/16.png)
![](/blog/images/obscurity/pics/user/17.png)
![](/blog/images/obscurity/pics/user/18.png)
![](/blog/images/obscurity/pics/user/19.png)
![](/blog/images/obscurity/pics/user/20.png)


Here we can see that it is a somewhat basic **python web server**. The biggest security issue that stands out is the use of the python **exec()** function. We find that it is accepting a string formatted **URL path**, that can be provided by us. We should be able to **escape the string formatting** with **_';_** and **inject some reverse shell python code** that will be executed by the **exec()** function. We will use a payload that we found on [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#python).

![](/blog/images/obscurity/pics/user/21.png)


This is the **reverse shell payload** that we will use, and we **url encode** it as well, getting it ready to send as a **GET Request**.

![](/blog/images/obscurity/pics/user/22.png)


We setup a local **netcat listener**.

![](/blog/images/obscurity/pics/user/23.png)


Then setup our **GET Request** with our payload in **Burp Suite**, and send the Request.

![](/blog/images/obscurity/pics/user/24.png)


We check our **netcat listener** and see that we got a **callback** and now have **shell access as www-data**. We now have a foothold in the system.

![](/blog/images/obscurity/pics/user/25.png)

***

## User / robert

After browsing around a bit we see some interesting files in the user **/home/robert/** directory. Here we see there is a **user.txt** file that we do not have permissions to read, as well as a python script called **SuperSecureCrypt.py**, and data files **passwordreminder.txt**, **check.txt**, and **out.txt**, as well as a directory named **BetterSSH** which we will investigate later.

![](/blog/images/obscurity/pics/user/33.png)


We **cat the files** to see what they contain.

![](/blog/images/obscurity/pics/user/27.png)


Now we look at the source code for **SuperSecureCrypt.py** and try to see what and how these files are used or generated.

![](/blog/images/obscurity/pics/user/30.png)
![](/blog/images/obscurity/pics/user/31.png)
![](/blog/images/obscurity/pics/user/32.png)


Looking at all this, it appears that the contents of **check.txt** was encrypted with a **secret key**, and it's output is in **out.txt**. After inspecting the algorithm used to encrypt, it looks like we should be able to reverse the encryption by simply using the **_-d_** switch and feeding the output (**out.txt**) as the input, and use the un-encrypted **check.txt** text as our key. This should output the original key used to encrypt **check.txt**, but repeating to match the character length of the original message.

![](/blog/images/obscurity/pics/user/35.png)


With **passwordreminder.txt** being an encrypted file as well, we see if the user used the same key, **alexandrovich**, to encrypt it.

![](/blog/images/obscurity/pics/user/37.png)


And this turns out to be the **password** for the account **robert**. So we **SSH** into the account using this password, and read the **user.txt** file to get the flag.

![](/blog/images/obscurity/pics/user/39.png)

***

## Privilege Escalation / root

Running **sudo -l** under this account shows us we have privileges to run **/usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py** as **sudo** with **no password**.

![](/blog/images/obscurity/pics/root/3.png)


We go back into that directory and look at the **source code** for this python program.

![](/blog/images/obscurity/pics/root/28.png)
![](/blog/images/obscurity/pics/root/29.png)


It looks like this program will ask you for your user credentials and validate them with **sudo** privileges from the **/etc/shadow** file. Once authorized you can take advantage of the hard coded **sudo** command it has by simply sending **_-u root_**, then your command, which will be appended to **_sudo_** in the highlighted lines above. With full **root privileges** now we can easily read the **/root/root.txt** file.

![](/blog/images/obscurity/pics/root/5.png)

With the ability to run any command with **sudo privileges** we could now create our own account or set up **SSH** for the **root** account and have full access to the machine indefinitely.

***


