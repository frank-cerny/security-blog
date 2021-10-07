---
layout: post
title:  "HTB: Sauna Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---

{:refdef: style="text-align: center;"}
![Sauna Logo](../../assets/img/writeups/htb/sauna/info_card.png)
{: refdef}

Date: 5/16/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`AS-REP Roasting -> Plaintext Password -> DCSync Privileges`

### Recon and Enumeration

As always, we started with a masscan, followed by a detailed nmap scan.

{:refdef: style="text-align: center;"}
![Detailed Port Scan](../../assets/img/writeups/htb/sauna/detailed_port_scan.png)
{: refdef}

The scan showed a multitude of known windows ports. Ther was ldap, kerberos, msrpc, etc. From this information, we could immediately tell this machine was a windows domain controller. Port 80 was also open, so we knew there was some form of webserver running as well.

The domain name nmap gave us was `egostical-bank.local0`

#### LDAP and Enum4linux

I always like to try `enum4linux` first. It encompasses a few information gathering methods into a single tool. Unfortunately, this time it could find almost no information about the domain.

{:refdef: style="text-align: center;"}
![Enum4Linux scan](../../assets/img/writeups/htb/sauna/enum4_linux.png)
{: refdef}

We then tried to search through ldap with `ldapsearch`. This did not give us much more infomration than `enum4linux` did, but we did get an updated domain name of `egotistical-bank.local`.

{:refdef: style="text-align: center;"}
![Ldap Search](../../assets/img/writeups/htb/sauna/ldap_search.png)
{: refdef}

After trying to get other ldap queries to work (to no avail), I decided to head over to the webserver.

#### Web Server

Gobuster did not find anything useful, regardless of the word list used. It was time for some manual enumeration. We were greeted with a homepage for a made up bank.

{:refdef: style="text-align: center;"}
![Webserver Homepage](../../assets/img/writeups/htb/sauna/http_1.png)
{: refdef}

I could find nothing useful in the source code, found no login forms, and found no other forms of user input. It was then I found the "Meet the Team" section at the bottom of the About page.

{:refdef: style="text-align: center;"}
![Meet the team page](../../assets/img/writeups/htb/sauna/meet_team.png)
{: refdef}

In the past, I usually breeze over employee names on the website, since they are mostly useless. Yet since I had found no other leads on this box so far, these names were intriguing. I thought that maybe these users were users in the domain. Piggybacking off what I learned while completing the Forest box a few months ago, if I could get a list of users in a domain, I might be able to steal a hashed password if the configuration was right. I took the names of the team members from the website, and created usernames for them.

Examples of usernames:

fergusSmith  
shaunCoin

### Initial Foothold and User

Now that we had a tentative list of users, I could use Impacket's GetNPUsers.py script to attempt to AS-REP Roast a user ([1]). See the section below on what AS-REP Roasting is for more details.

#### GetNPUsers

Some common problems I had while using this script:

Wrong Realm Error:

This error occurs if the domain you are trying to roast is not valid. At first I used `egotistical-bank.local0`, but got that error. Once I switched to `egotistical-bank.local` the error stopped.

KDC_ERR_C_PRINCIPAL_UNKNOWN:

The error occurs if the username supplied does NOT exist in the domain. This was helpful to determine if the usernames I created were right or not.

"User" doesn't have UF_DONT_REQUIRE_PREAUTH set:

This error occurs if the user is in the domain, but is not vulnerable to the attack

After getting PRINCIPAL_UNKNOWN errors for a few minutes, I found a username that caused a "is not vulnerable to this attack" error. This was helpful because it proved we found a valid username and username structure (even if they were not vulnerable to this attack). This structure was of course one of the most common structures out there. I wish I would have thought of it sooner.

Once we changed all usernames to be of the form \<first initial of first name\>\<last name\>, we found that `fsmith` was vulnerable to this attack, and we got a hash encrypted with his password.

{:refdef: style="text-align: center;"}
![AS-REP Roasting](../../assets/img/writeups/htb/sauna/get_np_users.png)
{: refdef}

#### What is AS-REP Roasting?

A quick aside here. The Kerberos protocol is all about tickets. Below is a rundown on an attack known as AS-REP Roasting:

1. Kerberos is based on tickets, handed out by a Key Distribution Center
2. To "apply" for tickets, a user must first be granted a "Ticket Granting Ticket" (TGT)
3. To get this TGT, the user must first authenticate with their password, since part of the response from the Key Distribution Center is encrypted (weakly) with this password
4. But, if DONT_REQ_PREAUTH is enabled (a configuration value in the user profile), we can get a TGT for a user without supplying a password
5. Thus, we can get a value that is known to be encrypted with the user's password, and can use hashcat or john to crack it and get back the password!

#### Hash Cracking

We then gave this hash to `john` and got back a password in less than a minute.

{:refdef: style="text-align: center;"}
![Cracking the hash with John](../../assets/img/writeups/htb/sauna/john.png)
{: refdef}

John cracked the hash, and gave us the password `Thestrokes23` for user `fsmith`.

If we recall the nmap scan, port 5985 (or Windows Remote Management) was open. This means we can use [evil-winrm](https://github.com/Hackplayers/evil-winrm){:target="blank"} to access the domain with these credentials ([2]). From there we can get the user hash and move on to privlege escalation.

{:refdef: style="text-align: center;"}
![Logging in with fsmith](../../assets/img/writeups/htb/sauna/user.png)
{: refdef}

### Privilege Escalation

#### WinPEAS

The first thing I like to do once I get remote access to a box is run either linPEAS or winPEAS. I ran winPEAS, and suprisingly I found a plaintext password for a user `svc_loanmgr`. Looks like it was in the WinLogon registry key, which seems to be used for auto logon capabilties. Wonderful for us.

{:refdef: style="text-align: center;"}
![Default Credentials](../../assets/img/writeups/htb/sauna/default_credentials.png)
{: refdef}

Using the `net user svc_loanmgr` command, we saw that `svc_loanmgr` was also a part of the Remote Management group. Hence we could login as them user via `evil-winrm`. Before switching to them though, I wanted to run `Bloodhound` and gather some permission data.

#### Bloodhound

Bloodhound enumerates the permission of each object in an Active Directory, and pinpoints where vulnerabilites may lie. I put `SharpHound.ps1` on the box by using a simple python server on my linux machine, and wget on the windows machine. The commands I used to run it were:

~~~powershell
Import-Module .\SharpHound.ps1
Invoke-BloodHound -Domain egotistical-bank.local -LDAPUser fsmith -LDAPPass Thestrokes23 -CollectionMethod All -ZipFileName test.zip
~~~

Then I ran `download test.zip` to get that zip file back onto my local machine. I ran Bloodhound with the following commands (in separate terminal screens):

~~~bash
neo4j console
bloodhound
~~~

With the GUI opened, I imported that data we got from the box. 

With the data in the system, we were able to query the data. We could have looked for things such as: 

- Quickest path to domain admin
- Quickest path to high value targets
- Etc.

The query that gave us the most information was "Show paths to high value targets". The only strange permissions I found was for the `svc_loanmgr` user. They had both the GetChanges, and GetChangesAll permission on the domain. 

{:refdef: style="text-align: center;"}
![Close up of svc_loanmgr permissions](../../assets/img/writeups/htb/sauna/svc_loanmanager_get_changes.png)
{: refdef}

A nice perk of Bloodhound is you can click on any permission to get more information about it. Clicking on either the GetChanges are GetChangesAll explains that when put together there can be trouble.

Looking at these permissions in more detail explained that when combined, it is equivilant to having DC-Sync privileges. Those privileges allow a user to dump hashes for all users in a domain, even administrators. Even better news was that we had login credentials for that user as well! See the section below for more information on what a DC-Sync attack is.

{:refdef: style="text-align: center;"}
![Bloodhound explaining GetChanges Vulnerability](../../assets/img/writeups/htb/sauna/get_changes_vuln.png)
{: refdef}

#### DC-Sync Explained

Note that most of my understaning comes from a video called [DC Sync Attacks With Secretsdump.py](https://www.youtube.com/watch?v=QfyZQDyeXjQ){:target="blank"} created by VbScrub on youtube. You should definitely watch it (and his other videos) to get a deeper understanding of why this is an issue ([4]). My understadning of this vulnerability is as follows:

1. Domain Controllers need to replicate the Active Directory amongst themselves to stay in sync
2. This includes users, passwords, computer information, etc.
3. The permissions needed to ask for a replicaton are: Replicating Directory Changes/Replicating Directory Changes All
4. We can grant the above permissions to a user
5. We can then ask another Domain Controller to replicate the Active Directory and hand it over to us
6. From there, we can grab the hashes it gave us and be on our way

#### DC-Sync in Action

There are multiple ways to execute this attack. I ended up using Impacket's `secretsdump.py` since I have had success with it before. I ran a DC-Sync as the user `svc_loanmgr` and got hashes for all users on the box, including the administrator.

{:refdef: style="text-align: center;"}
![Secrets Dump](../../assets/img/writeups/htb/sauna/secrets_dump.png)
{: refdef}

Using that hash, I used Impacket's `wmiexec` to login to the machine using the administrators password (pass the hash attack). 

{:refdef: style="text-align: center;"}
![Wmiexe root user](../../assets/img/writeups/htb/sauna/root_user.png)
{: refdef}

From there, I was able to get the root hash with ease.

{:refdef: style="text-align: center;"}
![Root hash](../../assets/img/writeups/htb/sauna/root_hash.png)
{: refdef}

### Conclusion

#### Fixes
1. Was able to AS-REP Roast `fsmith `  
   **Fix: Do not disable pre-authentication for kerberos on domain users**
2. Found a plaintext password for `svc-loanmgr` in the registry    
   **Fix: Do not store auto-login (or any type of login for that matter) credentials in the registry that can be read by low privileged users**
3. Was able to perform a DC-Sync attack and gain Administrative access to the machine    
   **Fix: Remove either the GetChanges or GetChangesAll permission from svc-loanmgr. Audit privileges to ensure no users can perform a DC-Sync. In general, no one should have any permission they do not need.**

#### Final Thoughts

I took a 2 month break in between user and root, but I am glad I came back to finish it. It refreshed my windows enumeration skills, and gave me tons of new material to brush up on. Thanks to `@egotisticalSW` and `@Helichopper` for helping me see what was right in front of me. Thanks for reading, and onto the next one.

*As always, if you have any questions or feedback, please contact me on HTB! I am always looking to improve my writing to make it as clear and concise as possible (while remaining somewhat beginner friendly). Happy Hacking!*

### References

1. Impacket library: https://github.com/SecureAuthCorp/impacket
2. Evil-winrm github: https://github.com/Hackplayers/evil-winrm
3. Bloodhound github: https://github.com/BloodHoundAD/BloodHound
4. VbScrub's DC Sync attack video on youtube: https://www.youtube.com/watch?v=QfyZQDyeXjQ
5. Microsoft GetChanges Documentation: https://docs.microsoft.com/en-us/windows/win32/adschema/r-ds-replication-get-changes?redirectedfrom=MSDN
6. Microsoft GetChangesAll Documentation: https://docs.microsoft.com/en-us/windows/win32/adschema/r-ds-replication-get-changes-all?redirectedfrom=MSDN