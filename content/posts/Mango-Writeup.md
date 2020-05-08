---
layout: post
title: "HackTheBox Mango Writeup"
categories: Writeup
date: 2020-04-17
description: Mango is a medium difficulty box where with basic enumeration and some MongoDB NO-SQL Injection we can extract user passwords to log in and get user access. From there we will leverage a classic jjs privilege escalation to get root access and read the root.txt file.
tags: [HTB, Writeup]
katex: true
markup: "mmark"
---

![](/images/mango/mango.png)

Mango is a medium difficulty box where with basic enumeration and some MongoDB NO-SQL Injection we can extract user passwords to log in and get user access. From there we will leverage a classic jjs privilege escalation to get root access and read the root.txt file.


## User
We first start off with an **nmap** scan of the machine IP Address. Then we convert our finished scan to an html file and view it in the browser.

![](/images/mango/pics/user/1.png)
![](/images/mango/pics/user/2.png)


Here we see there is an **Apache Web Server** running on **port 80** and **port 443**, and a listening **SSH Server** on **port 22**.

![](/images/mango/pics/user/3.png)
![](/images/mango/pics/user/4.png)


We now add those to our **/etc/hosts** file.

![](/images/mango/pics/user/5.png)


Next we check out the web pages in a browser. We see the one on **port 443** looks like some type of search engine, and has a user logged in.

![](/images/mango/pics/user/6.png)


So we have a look around in the few places that we can, which doesn't give us much to go on or do.

![](/images/mango/pics/user/7.png)


Next we see if there is any valuable information we can use in the **SSL Certificate**. Here we find a registered email address for this server. This could possibly be a **user name**.

![](/images/mango/pics/user/13.png)


We then check out the web page on **port 80** and find some sort of **login page**. We enter a test username and password, then intercept the request with **Burp Suite** to see what it looks like.

![](/images/mango/pics/user/10.png)
![](/images/mango/pics/user/9.png)


After a lot of googling and with the assumption that the name "Mango" may refer to "Mongo", as in **MongoDB**, we find that it may be vulnerable to a **NO_SQL Injection** attack, where we can fish out user passwords from the database. Here are some common [**MongoDB Payloads**](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/NoSQL%20Injection#mongodb-payloads).

We also find a [**script**](https://blog.0daylabs.com/2016/09/05/mongo-db-password-extraction-mmactf-100/) that may help us do this.

We try to guess a few usernames, including **admin** and **mango** and make a few edits to the script to get it working right. We were able to successfully extract the passwords for those two user accounts.

![](/images/mango/pics/user/12.png)
![](/images/mango/pics/user/14.png)
![](/images/mango/pics/user/15.png)


We then try to log into the web service with the two accounts. With the **admin** account we get this page as a response.

![](/images/mango/pics/user/16.png)


Next we try to long into the **SSH** server with the **admin** credentials but fail. So we try the **mango** credentials and successfully log into the machine.

![](/images/mango/pics/user/17.png)


After browsing around a bit we see that there is a **user.txt** file in **/home/admin/**, but we do not have access to read the file as our current user. So we try to **su** to the **admin** user with the credentials we have and are successful. We can now read the contents of **user.txt**.

![](/images/mango/pics/user/18.png)


## Root
To perform more enumeration on this machine, we start a simple python http server and use wget to download [**linPEAS**](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) from our host machine, then run the script.

![](/images/mango/pics/root/1.png)
![](/images/mango/pics/root/3.png)


**linPEAS** tells us that there is a very high probability that we may be able to leverage **jjs** to gain root access on this machine.

![](/images/mango/pics/root/4.png)


After a bit of googling we see there is a common [**privilege escalation vulnerability for jjs**](https://gtfobins.github.io/gtfobins/jjs/).

![](/images/mango/pics/root/5.png)


We execute this and are able to read the contents of **/root/root.txt**

![](/images/mango/pics/root/6.png)


With this exploit we also have the ability to write and download files with **root privileges**, giving us the option to create our own account or set up **SSH** for the **root** account, giving us full access to the machine indefinitely.
