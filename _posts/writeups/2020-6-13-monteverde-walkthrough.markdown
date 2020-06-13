---
layout: post
title:  "HTB: Monteverde Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---

{:refdef: style="text-align: center;"}
![Monteverde Logo](../../assets/img/writeups/htb/monteverde/info_card.png)
{: refdef}

Date: 6/13/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`Simple Password --> Plaintext password in SMB Share --> Azure AD Connect Exploit`

### Recon and Enumeration

As always, we start with a `masscan`, followed by a detailed `nmap` scan.

{:refdef: style="text-align: center;"}
![Detailed Port Scan](../../assets/img/writeups/htb/monteverde/detailed_port_scan.png)
{: refdef}

Given we have kerberos, ldap, and SMB, we were confident this machine was a domain controller. The domain name was `megabank.local0` (Similar to the domain in Resolute, another box by egre55). We added that domain name to our /etc/hosts file per usual.

#### enum4Linux

We gathered usernames and group memberships by running `enum4linux`.

{:refdef: style="text-align: center;"}
![enum4linux results](../../assets/img/writeups/htb/monteverde/enum4linux.png)
{: refdef}

We put all the usernames into a file for future use. We also notice there is a group called "Azure Administrators". [Azure](https://azure.microsoft.com/en-us/){:target="_blank"} is Microsoft's cloud platform and will likely be part of some CVE during this box.

#### Lazy Admins (Initial Foothold)

We spent the next two days looking for passwords. We tried AS-REP Roasting, anonymous SMB login, etc. Nothing seemed to be working. Multiple hints on the HTB Forums mentioned the password was easy to find due to laziness. I rarely brute force logins on boxes, so I thought about what kind of passwords I would set if I was a lazy admin. It finally hit me. Use the username as the password. One less thing to remember :)

I used `rpcclient` to test each username password combination, where the password was the username. I did not use `evil-winrm` for this, since it was likely that only a few users were part of the Remote Administrators Group (and could login remotely). 

Turns out our hunch was correct, and we found the username and password for `SABatchJobs` (password was also "SABatchJobs").

{:refdef: style="text-align: center;"}
![rpclogin](../../assets/img/writeups/htb/monteverde/rpcclogin.png)
{: refdef}

### Getting User

Since we were not able to login remotely (via `evil-winrm`) with SABatchJobs, we needed to find another way in. With valid credentials, I decided to check Samba again (using `smbclient`).

{:refdef: style="text-align: center;"}
![Authed SMB Client](../../assets/img/writeups/htb/monteverde/smbclient_authed.png)
{: refdef}

So we see there are shares available, but do not see our permissions on them. Using `smbmap`:

{:refdef: style="text-align: center;"}
![SMBMap Permissions](../../assets/img/writeups/htb/monteverde/smb_permissions_authed.png)
{: refdef}

Looks like we can read both azure_uploads and users$. I started with users$ since it seemed more interesting.

Following an answer on [Stack Overflow](https://unix.stackexchange.com/questions/99065/how-to-mount-a-windows-samba-windows-share-under-linux){:target="_blank"}, I mounted the users$ share on my local machine.

`mount -t cifs //10.10.10.172/users$ -o username=SABatchJobs,password=SABatchJobs,vers=2.0 /mnt/monte`

Navigating through the share, we find an Azure Config file in user `mhope's` home directory. Not suprisingly, it contains a plaintext password for `mhope`.

{:refdef: style="text-align: center;"}
![Mhope Azure XML](../../assets/img/writeups/htb/monteverde/mhope_azure_xml.png)
{: refdef}

Using this username and password combination, we can now use `evil-winrm` to login remotely and get the user hash.

{:refdef: style="text-align: center;"}
![Mhope User](../../assets/img/writeups/htb/monteverde/user_hope.png)
{: refdef}

We can see `mhope` is a member of the Azure Admins, which might be helpful later.

{:refdef: style="text-align: center;"}
![User Hash](../../assets/img/writeups/htb/monteverde/user_hash.png)
{: refdef}

### Privilege Escalation

It was clear to me that Azure Active Directory was the key to the privilge escalation portion of this box. I googled "Azure AD exploits" and was greeted with many articles. Most important were the ones by [VBScrub](https://vbscrub.video.blog/2020/01/14/azure-ad-connect-database-exploit-priv-esc/){:target="_blank"} (a fellow HTB member) and [Dirk-jan](https://dirkjanm.io/azure-ad-privilege-escalation-application-admin/){:target="_blank"}. Both contained information and links to an exploit called "Azure AD Connect Database Exploit".

The cliffnotes version of it (according to VBScrub):
1. When connected to a domain, Azure AD has a syncing tool (called Azure AD Connect)
2. This syncing tool syncs hashes, users, groups, etc
3. For this sync to occur, Azure AD Connect must have access to a privileged account on the domain
4. The credentials for this user are stored insecurely on the domain, and can be read/decrypted with "ease"

VBScrub made this much too easy for us. He layed out a guide on how to run a script he created (based loosely on Dirk-jan's original script). Upload his script to the box, make sure you are in the correct directory, and run it.

Exploit Path:
1. Downloaded VBScrub's script AdDecrypt.exe and mcrypt.dll
2. Uploaded both to the domain controller
3. Changed directory into `C:\Program Files\Microsoft Azure AD Sync\Bin`
4. Ran the program using the full SQL Server command*

*How did we know to use full SQL Server instead of a local one?*

`enum4linux` alerted us there was a SQL Server running on the domain controller. Even if we did not know, we could have tried both options of the payload. Running the program, we get the password for the Administrator. We can then login with psexec, and grab the root hash.

{:refdef: style="text-align: center;"}
![Running Exploit](../../assets/img/writeups/htb/monteverde/admin_sync.png)
{: refdef}

{:refdef: style="text-align: center;"}
![Root](../../assets/img/writeups/htb/monteverde/root.png)
{: refdef}

### Conclusion

#### Fixes

1. Username was password for SABatchJobs  
**Fix: Never use a username as a password for the same user**  
2. Password for mhope was stored in plaintext in an Azure Configuration File  
**Fix: Do not store credentials in plain text, or restrict the users$ samba share**
3. Abused Azure AD Connect to get Administrator password  
**Fix: Microsoft patched this specific vulnerability, but other similar ones still exist. Check out the articles in the references section for more details**  

#### Final Thoughts

I think I am finally starting to get a decent process down to attack windows boxes. I learned a few more techniques I can add to this process, that I think will make future Windows boxes even easier. Thanks to `egre55` for another good one and for `VBScrub` for a fantastic article. Onto the next one...

*As always, if you have any questions or feedback, please contact me on HTB! I am always looking to improve my writing to make it as clear and concise as possible (while remaining somewhat beginner friendly). Happy Hacking!*

### References

1. Microsoft Azure: https://azure.microsoft.com/en-us/
2. Moutning SMB Shares Stack Overflow Post: https://unix.stackexchange.com/questions/99065/how-to-mount-a-windows-samba-windows-share-under-linux
3. VBScrub Azure AD Connect Exploit Article: https://vbscrub.video.blog/2020/01/14/azure-ad-connect-database-exploit-priv-esc/
4. Dirk-jan's AD Connect Exploit Article: https://dirkjanm.io/azure-ad-privilege-escalation-application-admin/