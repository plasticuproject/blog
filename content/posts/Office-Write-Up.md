---
layout: post
title: "CyberSecLabs Office Write-Up"
categories: Write-Up
date: 2020-06-19
image: /blog/images/office/pics/CyberSecLabs.png
description: Office is a hard difficulty linux machine where we need to stop Dwight's "Accountability Booster" (Doomsday Device) by gaining admin access to a content management system and root access to a server via a system configuration tool vulnerability.
tags: [CyberSecLabs, Write-Up]
katex: true
markup: "mmark"
---

![CyberSecLabs](/blog/images/office/pics/CyberSecLabs.png#center)

---

## User / www-data and dwight

First we will run an **nmap** scan of the machine IP address, export our results to an HTML file, and view it in Firefox-ESR.

![nmap](/blog/images/office/pics/user/1.png)
![xsltproc](/blog/images/office/pics/user/2.png)
![](/blog/images/office/pics/user/3.png)
![](/blog/images/office/pics/user/4.png)

We see there is a **HTTP Web Server** running on **Port 80**, a **HTTPS Web Server** on **Port 443**, and a listening **SSH Server** on **Port 22**. Let's add the IP address and hostname to our **/etc/hosts** file.

![](/blog/images/office/pics/user/5.png)

Next let's take a look at the website on port 80.

![](/blog/images/office/pics/user/6.png)
![](/blog/images/office/pics/user/7.png)
![](/blog/images/office/pics/user/8.png)
![](/blog/images/office/pics/user/9.png)
![](/blog/images/office/pics/user/10.png)

Oh just another day at the Dunder Mifflin Office! It looks like Dwight has initiated some sort of "Doomsday Device" and is using a **WordPress** site and **Forum** to tell everyone in the office about it. We will soon see if there is any way we can get user credentials or bypass authentication.

![](/blog/images/office/pics/user/11.png)

Also when we take a look at the HTTPS server, we see it is just a default Apache landing page.

![](/blog/images/office/pics/user/12.png)

The SSL certificate also has **dwight@office.csl** as a listed email address. This may be a username we can try to use for the login.

![](/blog/images/office/pics/user/14.png#center)

Next let's try some directory fuzzing for our web servers. Not much turns up for the server on port 80, but the secure web server does have some interesting endpoints.

![](/blog/images/office/pics/user/15.png)
![](/blog/images/office/pics/user/16.png)

Let's take a look at the **forum** endpoint as see if we can find anymore useful information.

![](/blog/images/office/pics/user/17.png)
![](/blog/images/office/pics/user/18.png)

Looks like everyone in office is concerned about the Accountability Booster, each trying to find a way to deal with it. Some are even trying to brute force a log-in portal that was forwarded to Jim's local machine. We haven't found any such portal yet, but it's good to have an idea what to maybe look for later. Let's keep enumerating this site and check out what's on the **Login** and **Chat Logs** page.

![](/blog/images/office/pics/user/19.png)

We get an _Access Denied_ on the Login page, but the **Chat Logs** page does show us something interesting.

![](/blog/images/office/pics/user/20.png)

It appears to load and print out all of the chat messages we just read, from a **Text File**. So the web server is loading the contents of a local file using the **?file=** parameter. This looks like a great place to try a [Directory Traversal](https://www.acunetix.com/websitesecurity/directory-traversal/) exploit. Maybe we can get the server to print out the contents of a useful file.
![](/blog/images/office/pics/user/21.png)

Let's run **wfuzz** with the **LFI-Jhaddix.txt** wordlist found in the [SecLists Github Repository](https://github.com/danielmiessler/SecLists), trying each phrase as a filename in the URL parameter and see if we can find any useful files on the machine.

![](/blog/images/office/pics/user/22.png)
![](/blog/images/office/pics/user/23.png)

There seems to be a problem. It looks like wfuzz is returning a _200 OK HTTP Response Code_ on every filename that we try. That's not very useful as we won't know which files are really there or not. Luckily we can see that each response has a different amount of returned content, and can filter requests that are not useful to us based on the amount of data received. Looking at our output, it looks like anything with a line length of 27 may not actually be returning a valid file. Let's filter out any requests that return that number of lines and see what we have.

![](/blog/images/office/pics/user/24.png)
![](/blog/images/office/pics/user/25.png)

That's better. The first thing that catches my eye is **../.htpasswd**, which is a flat-file used to store usernames and passwords for basic authentication of HTTP users. Let's use that file as a parameter and see what it returns.

![](/blog/images/office/pics/user/26.png)

It contained a username/password hash combination for **dwight**! I will copy this hash over to another computer that has the hash cracker [JohnTheRipper](https://github.com/magnumripper/JohnTheRipper) installed and we will try to brute force crack this hash with the default wordlist.

![](/blog/images/office/pics/user/27.png)

And we were successful. We should now have log-in credentials for the WordPress login portal.

![](/blog/images/office/pics/user/28.png)

We are now successfully logged into the **WordPress Admin Dashboard**, and it looks like we have the ability to upload files. We will use this to get shell access to the machine with a [Local File Inclusion](https://www.acunetix.com/blog/articles/local-file-inclusion-lfi/) exploit, which will allow us to run the code in our uploaded file on the machine hosting the web server.

![](/blog/images/office/pics/user/29.png)
![](/blog/images/office/pics/user/30.png)

First we use the **ifconfig** command to find out what our network IP address is.

![](/blog/images/office/pics/user/31.png)

Now we find a [PHP Reverse Shell Script](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) and add our IP address to it, as well as a port number that we will set up a **netcat** listener on, for our script to call back to. Here we have renamed the script to **a.php**.

![](/blog/images/office/pics/user/32.png)

We start our netcat listener, and upload our reverse shell script.

![](/blog/images/office/pics/user/33.png)
![](/blog/images/office/pics/user/34.png)
![](/blog/images/office/pics/user/35.png)

To run our script we just append the filename to the web server URL.

![](/blog/images/office/pics/user/36.png)

And when we check our listener, we see that we received a callback and now have shell access to the machine.

![](/blog/images/office/pics/user/37.png)

We run the **whoami** command and see that we have access as user **www-data**, and with a little bit of poking around we find the flag file **/home/dwight/access.txt** and print out the contents.

![](/blog/images/office/pics/user/38.png)

When we run **sudo -l** we see that the **sudoers** file is configured to give our current user the ability to run the command **/bin/bash** with user **dwight** privileges, and is configured to do so without requiring a password using the **sudo** command. So running **sudo -u dwight /bin/bash** will effectively give us a **bash shell prompt** with dwight's privilege settings. After doing this we find dwight's **.ssh** directory and see that it is empty.

![](/blog/images/office/pics/user/39.png)

To get **ssh** login access to dwight on this machine, let's create a ssh key-pair on our host machine with **ssh-keygen**.

![](/blog/images/office/pics/user/40.png)

Then copy the **public key** that we created and paste it onto **/home/dwight/.ssh/authorized_keys** on the remote machine.

![](/blog/images/office/pics/user/41.png)

We now have **ssh** access to dwight on the remote machine with our private key file.

![](/blog/images/office/pics/user/42.png)

---

## Privilege Escalation / root

After browsing around the machine for a few minutes we find a file called **webmin-setup.out**.

![](/blog/images/office/pics/root/1.png)

Looking at this file, we can see the version number and port that this application is supposed to be running on.

![](/blog/images/office/pics/root/2.png)

Let's use **netstat** to see if there is actually anything running on this port.

![](/blog/images/office/pics/root/3.png)

We find that there is an application using this port, so it is most likely this **Webmin**. But what exactly is Webmin? 

![](/blog/images/office/pics/root/4.png)

[Wikipedia](https://en.wikipedia.org/wiki/Webmin) tells us that it is a web based system configuration tool for linux. Let's do a little bit of searching and see if the version of this software that is installed has any publicly known vulnerabilities.

![](/blog/images/office/pics/root/5.png)

It most definitely does have known vulnerabilities. Let's use ssh to forward this service port over to our local machine and take a look at it in our browser.

![](/blog/images/office/pics/root/6.png)
![](/blog/images/office/pics/root/7.png)

And here we have the **Log-in Portal** for the service. Is this the portal that Jim was talking about trying to find a password for? I'm not sure brute force guessing passwords is the best way to approach this, since it has some commonly known vulnerabilities. Let's start up a _Metasploit Framework Console_ with the **msfconsole** command on our host machine and search for exploit scripts for this service. We will try the **webmin_backdoor** exploit, as it seems that we do not need to be authenticated to successfully use this exploit.

![](/blog/images/office/pics/root/8.png)

We look at the options for the exploit and set the remote machine's address to our localhost address that we are forwarding the service too, then set our listening address to our host machine's network address.

![](/blog/images/office/pics/root/9.png)

Next we cross our fingers and run the exploit. Success! The exploit ran and called back to the listener, giving us an open session.

![](/blog/images/office/pics/root/10.png)

We now have a shell prompt with **root** level privileges and can print the flag file **/root/system.txt**

![](/blog/images/office/pics/root/11.png)

Now that we have root access to the machine, it would be trivial to stop Dwight's "Doomsday Device", or set up persistent access and do anything else we wanted to do. I will thank the creator for this very fun, creative challenge. I thoroughly enjoyed it.

---
