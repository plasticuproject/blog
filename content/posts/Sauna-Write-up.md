---
layout: post
title: "HackTheBox Sauna Write-Up"
categories: Write-Up
date: 2020-07-18
image: /blog/images/sauna/sauna.png
description: Sauna is an easy difficulty Windows machine where we exploit weaknesses in an Active Directory environment.
tags: [HTB, Write-Up]
katex: true
markup: "mmark"
---

![HackTheBox](/blog/images/sauna/sauna.png#center)

---

## User / jsmith

First we will run an **nmap** scan of the machine IP address, export our results to an HTML file, and view it in Firefox-ESR.

![](/blog/images/sauna/pics/user/1.png)
![](/blog/images/sauna/pics/user/3.png)
![](/blog/images/sauna/pics/user/4.png)
![](/blog/images/sauna/pics/user/5.png)

Here we can see the machine is running a **Web Server**, and a number of other services like **Kerberos**, **RPC**, and **Active Directory LDAP**. We do some light, manual recon on the web server and do not find much. One thing that did get my attention is an **about** page with a list of employees, some of which may have user accounts.

![](/blog/images/sauna/pics/user/10.png)
![](/blog/images/sauna/pics/user/11.png)

Let's run nmap again, this time using a [script](https://nmap.org/nsedoc/scripts/ldap-search.html) to try and enumerate the [LDAP](https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol).

![](/blog/images/sauna/pics/user/6.png)
![](/blog/images/sauna/pics/user/7.png)
![](/blog/images/sauna/pics/user/8.png)

Here we can see that there is a user named **Hugo Smith** and the **Domain Controller** is **EGOTISTICAL-BANK.LOCAL**. Mr. Smith is one of the employee names listed on the _about_ page. Before we move any further let's add the domain controller and web server names to our **/etc/hosts** file.

![](/blog/images/sauna/pics/user/13.png)

Trying different username formats for the users that we found, we try to use perform [AS-REP Roasting](https://www.hackingarticles.in/as-rep-roasting/) with [impacket-GetNPUsers.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/GetNPUsers.py), which will try to retrieve a user's **Kerberos AS-REP Password Hash** if that user has **_“Do not use Kerberos pre-authentication”_** set on their account.

![](/blog/images/sauna/pics/user/14.png)

We've successfully extracted the password hash for user **fsmith** by AS-REP Roasting. Now let's copy that hash to another computer that has [JohnTheRipper](https://github.com/magnumripper/JohnTheRipper) installed and try to crack the hash with the [rockyou.txt wordlist](https://github.com/danielmiessler/SecLists/tree/master/Passwords/Leaked-Databases).

![](/blog/images/sauna/pics/user/john.png)

John successfully cracked the hash. We still don't have a clear entry point for logging into this machine so let's run nmap again, this time using the _-p_ switch to look at all TCP ports.

![](/blog/images/sauna/pics/user/nmap.png)

Here we can see **Port 5985** is open. This port handles [WinRM (Windows Remote Management)](https://en.wikipedia.org/wiki/Windows_Remote_Management), an implementation of WS-Management (Web Services-Management) that can be authenticated with [Kerberos](https://en.wikipedia.org/wiki/Kerberos_%28protocol%29).
Now we can attempt to log in through WinRM with the credentials we obtained and cracked using AS-REP Roasting and JohnTheRipper. We will use [Evil-WinRM](https://github.com/Hackplayers/evil-winrm) to do this as it will give us the some added functionality, like the ability to upload and download files between our host and client very easily. After we log in we will find and print the **user.txt** file.

![](/blog/images/sauna/pics/user/15.png)


---

## Privilege Escalation / Administrator

We run _net user_ and get a list of all user accounts. We will ultimately want to get access to the Administrator account, but we may need to pivot through another account to do so.

![](/blog/images/sauna/pics/root/6.png)

We can upload and run the [WinPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/winPEAS) enumeration script. This will help us discover user information, privilege settings, groups, stored credentials, and show us other things that may help us find vulnerabilities for privilege escalation.

![](/blog/images/sauna/pics/root/5.png)

WinPEAS shows us some valuable information about our current user. It has also found **stored credentials** for the **svc_loanmanager/svc_loanmgr** account.

![](/blog/images/sauna/pics/root/n3.png)
![](/blog/images/sauna/pics/root/n4.png)
![](/blog/images/sauna/pics/root/n5.png)
![](/blog/images/sauna/pics/root/4.png)

Next we'll use the credentials that we found to log in as **scv_loanmgr**. 

![](/blog/images/sauna/pics/root/7.png)

We run our WinPEAS script again, but don't really discover much of anything else. At this point I think it is a good idea to run [BloodHound](https://github.com/BloodHoundAD/BloodHound), an Active Directory reconnaissance tool. We can use BloodHound to identify different attack paths through a visualized graph. [Here](https://www.pentestpartners.com/security-blog/bloodhound-walkthrough-a-tool-for-many-tradecrafts/) is a good article explaining how to download, install, and get started using BloodHound. After we get everything installed and working we will upload the SharpHound Ingestor script, run it with the appropriate arguments, and download the zipped results. At this time we will also upload [mimikatz](https://github.com/gentilkiwi/mimikatz), which we will talk about later.

![](/blog/images/sauna/pics/root/16.png)
![](/blog/images/sauna/pics/root/8.png)
![](/blog/images/sauna/pics/root/18.png)

Afterwords we will download the data, import it into BloodHound and start looking around. Here is the shortest path from svc_loanmgr to Domain Admin.

![](/blog/images/sauna/pics/root/21.png)

One of the first things we look at is if any users can use a [DCSync attack](https://qomplx.com/kerberos_dcsync_attacks_explained/), which will allow them to simulate the behavior of a Domain Controller and ask other Domain Controllers to replicate information using the [Directory Replication Service Remote Protocol (MS-DRSR)](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/06205d97-30da-4fdc-a276-3fd831b272e0). By searching _Find Principals With DCSync Rights_ we find that our current user, svc_loanmgr, should be able to perform this attack.

![](/blog/images/sauna/pics/root/19.png)

We can try running the **mimikatz** program we uploaded earlier, selecting the [DCSync](https://attack.stealthbits.com/privilege-escalation-using-mimikatz-dcsync) option, using it to dump the NTLM Hash for the **Administrator** account.

![](/blog/images/sauna/pics/root/9.png)

We now have Administrator account ownership, can log in through WinRM, and print the **root.txt** file. With this exploit we can also get the [KRBTGT](https://adsecurity.org/?p=483) password hash, giving us the ability to create [Golden Tickets](https://www.qomplx.com/qomplx-knowledge-golden-ticket-attacks-explained/) which will grant us unfettered access to the network.

![](/blog/images/sauna/pics/root/13.png)

I don't have much experience with Windows Active Directory attacks, so this was a great machine for getting started and learning some basic principles and methods. I very much enjoyed the process.




---


