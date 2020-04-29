---
layout: post
title:  "HTB: Forest Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---

![Forest Logo](../../assets/img/writeups/htb/forest/info_card.png)

Date: ?/?/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`AS-REP Roasting --> Remote Shell with WinRM --> DCSync Replication privlidges on domain --> Administrator Access`

*Disclaimer. This was my first windows box, so my knowledge in this area is lacking. I have attempted to research all claims I make about how and why something works in this box. Yet, I understand I will be wrong in some of those claims.*

### Recon and Enumeration

Per usual, we ran `masscan`, then ran a detailed nmap scan on the open ports. This box had a TON of open ports. Shown below is a detailed scan on the more important ones (really anything under 10000).

{:refdef: style="text-align: center;"}
![TCP Scan](../../assets/img/writeups/htb/forest/detailed_tcp_scan.png)
{: refdef}

It took me some time to realize this box was a domain controller. The central "controller" of a windows active directory environment. This was evident because certain services like `kerberos` and `ldap` were running. So which ports are important?

- 88 (Kerberos)
- 389 (ldap)
- 445 (samba)
- 5985 (Windows Remote Management)

I checked for any port that could be running a webserver, but could not find anything. I first checked if there were any samba shares available. No matter which tool I used (smbclient, nmap, etc.) no open samba shares could be found. Onto ldap then!

{:refdef: style="text-align: center;"}
![Samba Access Denied](../../assets/img/writeups/htb/forest/no_smb_map.png)
{: refdef}

#### Getting Usernames with Enum4Linux

There are multiple ways to find the information below (ldapsearch, rpcclient, etc) but we will show how we used `enum4linux` since that is what I used originally (on a hint from a friend on the forums).

We ran `enum4linux 10.10.10.161`. It outputs ton of information, but the most important information was a user list.

{:refdef: style="text-align: center;"}
![Enum4Linux user list](../../assets/img/writeups/htb/forest/enum_4_linux_users.png)
{: refdef}

The one user that sticks out is `svc-alfresco`. We will use that insight later. Other useful inforamtion it gave us was the domain name of `HTB`. 

We wrote those usernames to a list, as we thought it might come in handy later....

#### AS-REP Roasting

After some more hints from the forum on using Impacket, I read old writeups on Active and Sizzle. Those combined with the hints led me to the Impacket script `GetNPUsers.py`. It is my belief that this script attempts to search for users that have the DONT_REQ_PREAUTH property. 

Quick Rundown on how this works:
1. Kerberos is based on tickets, handed out by a Key Distribution Center
2. To "apply" for tickets, a user must first be granted a "Ticket Granting Ticket" (TGT)
3. To get this TGT, the user must first authenticate with their password, since part of the response from the Key Distribution Center is encrypted (weakly) with this password
4. But, if DONT_REQ_PREAUTH is present, we can get a TGT for a user without supplying a password. 
5. Thus, we can get a hash that is known to be encrypted with the user's password, and can use hashcat or john to crack it and get back the password!

We ran the script that searches for these hashes that we can crack. Note that users.txt contains the users we found above.

`python3 GetNPUsers.py HTB / -userfile users.txt -format john -outputfile hashes -no-pass`

We find that svc-alfresco is vulnerable to this attack, and we get a hash back for that account!

{:refdef: style="text-align: center;"}
![AS-REP Roasting](../../assets/img/writeups/htb/forest/as_rep_roast.png)
{: refdef}

Putting this hash in a file, we let john do its magic.

{:refdef: style="text-align: center;"}
![John the Ripper](../../assets/img/writeups/htb/forest/hash_cracking.png)
{: refdef}

We now have credentials for a user on the domain! 

### Initial Foothold (and User)

I tried to connect via the windows command prompt, but stuck with linux since I could not get the windows methods to work (although I am sure that they exist)

The only other port that seemed of interest was Windows Remote Management. More information can be found on [Microsoft's Website](https://docs.microsoft.com/en-us/windows/win32/winrm/portal){:target="_blank"}([3]). Essentially, the service allows different hardware and OS's to communicate. It did not seem vulnerable to me.

Googling "windows remote management exploit" returned a few results. Turns out there is a metasploit module based around the service. Unfortunately, I could not get it to work. The 3rd search result was a Github project titled "evil-winrm" by Hackplayers. The first line of the README states, "This shell is the ultimate WinRM shell for hacking/pentesting". It gives us a wealth of capability, BUT only if we have authentication credentials... See the full respository [here](https://github.com/Hackplayers/evil-winrm){:target="_blank"}([4]).

After installing `evil-winrm` we can supply the credentials we found above to get a remote shell to the domain controller.

{:refdef: style="text-align: center;"}
![Evil WinRm](../../assets/img/writeups/htb/forest/alfresco_user.png)
{: refdef}

This not only provided a foothold, but a user. We found the user hash in svc-alfresco\Desktop

{:refdef: style="text-align: center;"}
![User hash](../../assets/img/writeups/htb/forest/user_text.png)
{: refdef}

### Privilege Escalation

#### Bloodhound (Walk the Dog)

This box relies heavily on active directory. I could not think of a better tool than `Bloodhound` to help us enumerate that. In essence, the tool helps enumerate and visualize the relationships between objects in active directory. See the [repository](https://github.com/BloodHoundAD/BloodHound){:target="_blank"} for more information ([5]).

After installing Bloodhound and its neccessary dependencies, I uploaded SharpHound.ps1 (a data "ingestor") to the Downloads folder of svc-alfresco. An "ingestor" can be thought of as a master enumerator, that collects data so we can view it later. The key here is that the ingestor will not return any data (nor an error) unless we provide it the correct parameters. Most people I talked to forgot to include the credentials of the user!

{:refdef: style="text-align: center;"}
![Sharphound Run](../../assets/img/writeups/htb/forest/sharphound_run.png)
{: refdef}

With the data in tow, lets pop it in Bloodhound for some analysis (follow the guide on the github page for how to do this). We will use a prebuilt query from Bloodhound by clicking the 3 Horizontal Lines at the stop left (seen in the picture below), selecting "Queries" then selecting "Shortest Path to Domain Admin"

{:refdef: style="text-align: center;"}
![Bloodhound Query Help](../../assets/img/writeups/htb/forest/bloodhound_queries.png)
{: refdef}

The graph it brings up can be seen below. Pay very close attention to the crudely highlighted relationships between nodes.

{:refdef: style="text-align: center;"}
![Detailed Graph](../../assets/img/writeups/htb/forest/graph_detailed.png)
{: refdef}

So what does this graph tell us?

1. svc-alfresco is a member of the Service Account Group
2. Which in turn has membership in the Priviledged IT Account Group
3. Which in turn has membership in the Account Operators Group
4. Which has "Generic All" permissions on Exchange Windows Permissions Group
5. Which has "WriteDACL" permissions on the domain

*What are Generic All permissions?*

Full rights to the object (eg. Add users to a group, reset a password, etc)

*What are WriteDACL permissions?*

Can modify the ACL (Access Control Lists) of an object

Thanks for [ired.team](https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/abusing-active-directory-acls-aces){:target="_blank"} for quick definitions on these ([6]).

#### New User Who Dis?

Since `svc-alfresco` is a member of the Account Operators Group by transitivity, we can create a new user in the Exchange Windows Permissions Group. **While the rest of the exploit works just fine by using svc-alfresco, it can make the box much more difficult for other people**. We ran the following commands through `evil-winrm` as `svc-alfresco`.

~~~powershell
net user lamp Changethis /add
net group "Exchange Windows Permissions" lamp /add
# So we can use evil-winrm to connect with them (thanks to @Radixx for this hint)
net localgroup "Remote Management Users" lamp /add
~~~

Now we can login with this user. I reran sharphound and generated a new graph with my new user on it. This will come in handy for the next part.

#### DCSync

It's all fun and games creating new users, but the real vulnerability lies in the "Write DACL" permissions that the Exchange Windows Permissions Group has on the domain. A simple google of "Exchange Windows Permissions WriteDACL" returns a wealth of results. I found the articles by [dirk-jam](https://dirkjanm.io/abusing-exchange-one-api-call-away-from-domain-admin/){:target="_blank"} and [gedegrous](https://github.com/gdedrouas/Exchange-AD-Privesc/blob/master/DomainObject/DomainObject.md){:target="_blank"} to be the most informative ([9,10]). 

*So what is the vulnerability?*

According to `dirk-jam`, since we have WriteDACL permissions on the domain, we can give any user DCSync rights, **"Users or computers with this privilege can perform synchronization operations that are normally used by Domain Controllers to replicate, which allows attackers to synchronize all the hashed passwords of users in the Active Directory"** ([9]).

So if we can give our user DCSync rights, we can get the NTLM hash of the administrator, and log in as them!!

#### Aclpwn

To get DCSync rights, a friend of my suggested I take a look at [aclpwn](https://github.com/fox-it/aclpwn.py){:target="_blank"}([7]). This tool automates escalating ACLs so we do not have to manually change them oursevles. **There are other tools and methods that do this, this is just the one I chose.** While I could have done this manually, my Windows abilities are not at that level yet. Following the github page (and after installation) we run aclpwn at the same time our newly updated graph is sitting in Bloodhound.

{:refdef: style="text-align: center;"}
![AclPwn Run](../../assets/img/writeups/htb/forest/acl_pwn_run.png)
{: refdef}

There is a lot going on here. Let's break it down.

1. -f lamp -t HTB.LOCAL ==> Look for paths between the nodes lamp and HTB.LOCAL (as seen in the Bloodhound Graph)
2. -d is the domain in question
3. AclPwn then automatically finds the path of escalation, and changes permissions so lamp now has DCSync rights on the domain

Now that we have DCSync rights, we can use Impackets secretsdump.py to dump the administrator hash (and many others).

{:refdef: style="text-align: center;"}
![Impacket Hash Dump](../../assets/img/writeups/htb/forest/secrets_dump.png)
{: refdef}

#### Pass the Hash

Now that we have the admin NTLM hash, we can "Pass the Hash" and login as admin. I found out about this tool by reading this [blog post](https://snowscan.io/htb-writeup-sizzle/#){:target="_blank"} on an old box called Sizzle ([8]). The tool in question is called wmiexec, and is from Impacket (of course).

{:refdef: style="text-align: center;"}
![Impacket Pass the Hash](../../assets/img/writeups/htb/forest/admin_shell.png)
{: refdef}

And with that, we are done.

### Conclusion

#### Fixes

**Following my disclaimer at the top of the page, these are suggested fixes that I think will work, there is 0 guarantee my guess is correct**

1. We were able to get a TGT for `svc-alfresco` and get his password from it  
**Fix: Use a password longer than 7 characters and DO NOT allow DONT_REQ_PREAUTH to be set**
2. We were able to get a shell on the machine using `evil-winrm`  
**Fix: Do not run services that are not being used (no easy way to tell if this service was actually needed on the box)**
3. We were able to create a new user in the Exchange Windows Permissions Group  
**Fix: Audit permissions to ensure low level accounts do not have more permissions than neccessary**
4. We were able to grant ourselves DCSync rights and get the admin hash  
**Fix: Reading some of `dirk-jams` article, it seems this no longer is possible by default, but is possible if the domain is misconfigured. Again, auditing permissions is key to ensure no user or group has permissions they do NOT need.**

#### Final Thoughts

This was arguably the most challenging box I have ever worked on. In the end though, the path of exploitation was relatively straightforward. Being my first windows box, I learned enough for a few years. I am excited to try other windows boxes and apply some of the things I learned here. Shoutout to `Masashig3` for heping me get started on the box, and to `egre55`/`mrb3n` for the creation of the box. Onto the next one...

*As always, if you have any questions or feedback, please contact me on HTB! I am always looking to improve my writing to make it as clear and concise as possible (while remaining somewhat beginner friendly). Happy Hacking!*

### References
1. HTB: Active Walkthrough: https://medium.com/bugbountywriteup/active-a-kerberos-and-active-directory-hackthebox-walkthrough-fed9bf755d15
2. HTB: Sizzle Walkthrough: https://0xrick.github.io/hack-the-box/sizzle/
3. Microsoft WinRM Page: https://docs.microsoft.com/en-us/windows/win32/winrm/portal
4. Evil-WInRm Github: https://github.com/Hackplayers/evil-winrm
5. Blodhound Github: https://github.com/BloodHoundAD/BloodHound
6. Ired.team Kerberos: https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/abusing-active-directory-acls-aces
7. AclPwn Github: https://github.com/fox-it/aclpwn.py
8. Sizzle Walktrhough: https://snowscan.io/htb-writeup-sizzle/#
9. DCSync Vulnerability: https://dirkjanm.io/abusing-exchange-one-api-call-away-from-domain-admin/
10. Exchange AD Vulnerability: https://github.com/gdedrouas/Exchange-AD-Privesc/blob/master/DomainObject/DomainObject.md