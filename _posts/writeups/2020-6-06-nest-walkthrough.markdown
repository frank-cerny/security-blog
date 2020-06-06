---
layout: post
title:  "HTB: Nest Walkthrough"
categories: writeups HTB
permalink: /:categories/:title
---

{:refdef: style="text-align: center;"}
![Nest Logo](../../assets/img/writeups/htb/nest/info_card.png)
{: refdef}

Date: 6/6/2020   
Author: n0tAc0p  

Table of Contents
1. TOC 
{:toc} 

### TL;DR

`Plaintext credentials for TempUser from Anonymous Login to SMB --> Password for C.Smith from encrypted "hash" and RU Scanner Program --> HQK Reporting debug password hidden in 0 length file (Alternate Data Streams) --> Admin password decrypted using c# .exe found in C.Smith share`

This box was an absolute beast. I required quite a few hints from the forums to get through it. It was almost pure enumeration. No CVEs, just information gathering and "simple" exploits. One of my favorite boxes to date. Let's hop into it.

### Recon and Enumeration

As always, I started with a `masscan` and an `nmap` scan. To my suprise, the machine was not a domain controller!

{:refdef: style="text-align: center;"}
![Detailed nmap scan](../../assets/img/writeups/htb/nest/detailed_port_scan.png)
{: refdef}

Port 445 was labeled as "Microsoft DS" by nmap, but from experience I knew this is the port that runs SMB (Network Storage/Shares) over TCP. The other port (4386) was labeled as an unknown service by `nmap` but based on the results it returned, it is called Reporting Service v1.2. I had 0 clue what that was. 

### Initial Foothold

Since I have more experience with Samba, I decided to start there.

#### SMB Enumeration

The first thing I did was run `smbmap` with an anonymous login, to see if we could view any of the shares as an unauthenticated user. 

{:refdef: style="text-align: center;"}
![Anonymous smbmap](../../assets/img/writeups/htb/nest/anonymous_smbmap.png)
{: refdef}

We had unauthenticated access! We had read access to the Users and Data share. I decided to try and look through the Data share first, since I most likely had no access for the User share (that turned out to be a correct assumption).

I used `smbclient` to connect to the Data share.

**Note: if you use the -R flag, you will get a tree view of all the files/folders you can access. I learned about this option during this box, but did not use it originally**

When asked for a password, I did not enter one.

{:refdef: style="text-align: center;"}
![Anonymous Share Access](../../assets/img/writeups/htb/nest/anonymous_share_access.png)
{: refdef}

I looked around the share, and read every file I could. There was not many to go through thankfully. Eventually, I found a filed called "Welcome Email" at `\Shared\Templates\HR\Welcome Email`. This email contained credentials for a user named `TempUser`. It looks like an email the user would have recieved on their first day.

{:refdef: style="text-align: center;"}
![Welcome Email](../../assets/img/writeups/htb/nest/welcome_email.png)
{: refdef}

Now I had potential credentials `TempUser:welcome2019`. I thought no better place to test if they worked than SMB :).

#### Authenticated SMB Enumeration

The credentials worked, and I was able to authenticate with SMB and gain access to more shares. This time using the -R flag in `smbmap` so I could see which files I could access immediately, instead of manually searching.

{:refdef: style="text-align: center;"}
![TempUser Share Access](../../assets/img/writeups/htb/nest/tempUser_smb_permissions.png)
{: refdef}

Looks like we gained access to the Secure$ share. Since I used the -R option, I was able to see all the files this user could access from a high level. I dug deeper into the more interesting looking ones and found a few things:

1. A empty text file in the TempUser share, and it seemed to serve no purpose
2. Some form of a password for C.Smith and another file which referenced his username (in Data\IT)

#### C.Smith "Hash"

I found a file called `RU_config.xml` at `\Data\IT\Configs\RU_Scanner\RU_config.xml`. It listed a username of c.smith and a password that looked to be base64 encoded.

{:refdef: style="text-align: center;"}
![C.Smith Hash](../../assets/img/writeups/htb/nest/ru_config.png)
{: refdef}

I was so excited I found a password, I immediately tried to authenticate with C.Smith over SMB. Didn't work. I tried to base64 decode it, determine the type of hash, etc. Nothing. I turned to the forums and a comment from `TazWake` made me realize that the password was encrypted with some custom program (something right under my node). I redoubled my efforts to read all the files I could (mostly config files). That is when I came across a `notepad++` file which mentioned c.smith.

{:refdef: style="text-align: center;"}
![Notepad History](../../assets/img/writeups/htb/nest/notepad_history.png)
{: refdef}

I found that file at `\Data\IT\Configs\notepadplusplus\config.xml`

The file mentions 3 other files:

1. A hosts file (which we cannot access and seemed unimportant)
2. A folder in the Secure$ share that we could not see since we could not access the IT folder
3. A todo file on C.Smith's desktop (which we could not access at this point)

Even though we did not have access to the `Secure$\IT` folder, I thought it would be a good idea to check if we could access `Secure$\IT\Carl`. Permissions are hard, and if they were misconfigured, we might be able to access it. I connected to the Secure$ share with TempUser, then tried to access `IT\Carl`. To my suprise, we were able to see the files in that folder! See below for the reasoning this worked.

The folder contained a few other folders.

{:refdef: style="text-align: center;"}
![Carl's "Hidden" folder](../../assets/img/writeups/htb/nest/carl_folder.png)
{: refdef}


At this point, I decided to mount the drive locally, so I could move these files to my local machine to use. I created a folder at `/mnt/nest/carl` and ran the following command:

`mount -t cifs -o user=TempUser //10.10.10.178/Secure$ /mnt/nest/carl`

I learned about this syntax from [the Samba wiki](https://wiki.samba.org/index.php/Mounting_samba_shares_from_a_unix_client){:target="blank"} ([1]).

I then copied all the files in the Carl folder to a local folder for inspection. Upon initial inspection, it seemed like the RU_Scanner project (Inside the VB Projects\WIP folder) was a visual basic project that could encrypt and decrypt text. Since we found C.Smith's encrypted password inside a file called `RU_Config.xml` there was a high chance this code did that encryption. Now all I had to do was make it decrypt the value.

#### Welcome to VB Town

Since I was runnning Kali, and did not have enough space to download Visual Studio I used [dotnetfiddle](https://dotnetfiddle.net/){:target="blank"} to compile the project ([4]). This was based on a hint in the forums from the boxes creator `VbScrub`. Before I found that hint, I tried a few other online compilers, but ran into issues with the AESCryptoServiceProvider not being defined. I think this had to do with the version of the .NET library those websites were using.

The project itself was mostly straightforward. I focused on the file `utils.vb` since it contained the logic to encrypt and decrypt text. I removed everything except the functions that dealt with decryption (since everything else was uneeded at this point). I added a main method which called the decrypt function with C.Smith's encrypted password and let it run. 

My Code:

~~~vb
Imports System
Imports System.Text
Imports System.Security.Cryptography

Public Module Driver

	Public Function Main()
		Console.WriteLine("C.Smith's Password is: {0}", DecryptString("fTEzAfYDoz1YzkqhQkH6GQFYKp1XY5hm7bjOP86yYxE="))
	End Function

	Function DecryptString(EncryptedString As String) As String
		If String.IsNullOrEmpty(EncryptedString) Then
			Return String.Empty
		Else
			Return Decrypt(EncryptedString, "N3st22", "88552299", 2, "464R5DFA5DL6LE28", 256)
		End If
	End Function

	Function Decrypt(ByVal cipherText As String, ByVal passPhrase As String, ByVal saltValue As String, ByVal passwordIterations As Integer, ByVal initVector As String, ByVal keySize As Integer) As String
		Dim initVectorBytes As Byte()
		initVectorBytes = Encoding.ASCII.GetBytes(initVector)
		Dim saltValueBytes As Byte()
		saltValueBytes = Encoding.ASCII.GetBytes(saltValue)
		Dim cipherTextBytes As Byte()
		cipherTextBytes = Convert.FromBase64String(cipherText)
		Dim password As New Rfc2898DeriveBytes(passPhrase, saltValueBytes, passwordIterations)
		Dim keyBytes As Byte()
		keyBytes = password.GetBytes(CInt(keySize / 8))
		Dim symmetricKey As New AesCryptoServiceProvider
		symmetricKey.Mode = CipherMode.CBC
		Dim decryptor As ICryptoTransform
		decryptor = symmetricKey.CreateDecryptor(keyBytes, initVectorBytes)
		Dim memoryStream As IO.MemoryStream
		memoryStream = New IO.MemoryStream(cipherTextBytes)
		Dim cryptoStream As CryptoStream
		cryptoStream = New CryptoStream(memoryStream, decryptor, CryptoStreamMode.Read)
		Dim plainTextBytes As Byte()
		ReDim plainTextBytes(cipherTextBytes.Length)
		Dim decryptedByteCount As Integer
		decryptedByteCount = cryptoStream.Read(plainTextBytes, 0, plainTextBytes.Length)
		memoryStream.Close()
		cryptoStream.Close()
		Dim plainText As String
		plainText = Encoding.ASCII.GetString(plainTextBytes, 0, decryptedByteCount)
		Return plainText
	End Function
End Module
~~~

To my joy, the program output the plaintext password of `xRxRxPANCAK3SxRxRx`.

With  C.Smith's password in tow, it was time to head back to SMB (and to get the user hash).

#### Aside: Folder Permissions are Hard

So why was I able to view Carl's folder, but not the IT folder? You would think if I did not have access to the parent folder, I would not be able to access the child folder either. This all comes to down to ACE's (Access Control Entities) which make up the ACL (Access Control List) of a file or folder. An ACL is essentially a list of permissions that exist on a folder or file (at least in this case). ACL's are the basis of access control in Windows. After I rooted the box, I went back and looked at the ACL of each folder, to try and understand what was going on. I used CMD through `psexec`.

If we list the ACL of the IT folder:

{:refdef: style="text-align: center;"}
![IT ACLs](../../assets/img/writeups/htb/nest/it_icacls.png)
{: refdef}

To learn more about what some of these fields meant, I used an article by [Varonis](https://www.varonis.com/blog/ntfs-permissions-vs-share/){:target="blank"}, and the official documentation from [Microsoft](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/icacls){:target="blank"} ([2,3]).

The permissions in the picture above do NOT include TempUser, or any group they are apart of. That is the reason why I could not access or list the contents of the directory directly.

Now if we list the ACL of the Carl folder:

{:refdef: style="text-align: center;"}
![Carl ACLs](../../assets/img/writeups/htb/nest/carl_icacls.png)
{: refdef}

Here we see the Users group has all inhertied permissions from it's parent (which in this case are no permissions) and also the RX permission. If we read the Microsoft documentataion above, we see this stands for Read and Execute permission. This permission allows us to read/list files in the directory and execute any of them. So even though we do not have access to the parent folder, we have almost full access to the child directory :).

### Getting User 

#### SMB.... Again :)

New credentials potentially means more permissions. I ran `smbmap` recursively again and found a few new items to make note of.

{:refdef: style="text-align: center;"}
![C.Smith SMB Enumeration](../../assets/img/writeups/htb/nest/c.smith_smb_enumeration_1.png)
{: refdef}

I found multiple files that referenced an HQK Reporting Service. Recalling the original `nmap` scan, port 4386 mentioned something about a Reporting Service. Leaving these new files alone for a bit, I decided to check out this Reporting Service running at port 4386.

Before that though, I used `smbclient` to get the user hash from `\Users\C.Smith\user.txt`.

### Privilege Escalation

#### HQK Reporting Service

When I initially did the box I did basic enumeration on this service while working with SMB. I was able to interface with the service using `telnet`. I spent wayyyyy too much time trying to use netcat to connect. Once connected via `telnet` I ran the `help` command to see what other commands we could use.

{:refdef: style="text-align: center;"}
![HQK Reporting Service Non-Debug](../../assets/img/writeups/htb/nest/telnet_connection.png)
{: refdef}

After some playing around I determined:

1. `List` listed all files in a directory
2. `SETDIR` let us change directories
3. `RUNQUERY` supposed to run a query file to generate a report, but the system threw an error when I tried to run any query
4. `DEBUG` turned on some form of debug mode, no password I had worked here
5. `HELP` gave us a list of commands, or more info on any other command

So what actually was this service? Based on some googling I found a RCE CVE from [MDSec](https://www.mdsec.co.uk/2020/02/cve-2020-0618-rce-in-sql-server-reporting-services-ssrs/){:target="blank"} and some limited information from [Microsoft](https://docs.microsoft.com/en-us/sql/reporting-services/create-deploy-and-manage-mobile-and-paginated-reports?view=sql-server-ver15){:target="blank"} ([5,6]).

While this CVE did not affect this machine, I did learn more about the service. The service allowed reports to be generated based on a SQL database. These reports were based on queries you could write, and execute (hence the `RUNQUERY` command). HQK stands for "Hibernate Query Language". In all honestly, I found it difficult to find any concrete information on how to use this service (besides what the `Help` command provided me). I am interested to see how others found useful information. Still, the service seemed to be useless except for the debug mode. Of course, we did not have a password. Or did we?

#### Empty Files and ADS

Going back to the new files we found in SMB with C.Smith. We found an exe, a backup file (which contained nothing useful) and a file called "Debug Mode Password.txt". Easy, we found our missing debug password! Except the file was empty and had a length of 0 :(. I had read the forums earlier in the day and saw mention of "Alternate Data Streams". I googled around a bit and found an article by [Microsoft](https://docs.microsoft.com/en-us/sysinternals/downloads/streams){:target="blank"} explaining a tool used to deal with streams, and also an article from [Malwarebytes](https://blog.malwarebytes.com/101/2015/07/introduction-to-alternate-data-streams/){:target="blank"} explaining the basics of Alternate Data Streams ([7,8]). To sum up both these articles:

- Every file has a single "unnamed" stream which contains the data you see when you open the file in say, `notepad` or something similar
- We can write to any number of alternate "named" streams (using CMD or PowerShell)
- The data in these alternate named streams do not show up in a file's size, and are not shown as part of the unamed stream
- To read from an alternate stream, we need the stream name
- These alternate streams are commonly used to support software or other functions on Windows (ie. they are not usually malicious or sneaky)

Now that we have a basic understaninding of streams, we had to figure out how download a files alternate stream over SMB as well as find the stream name. Unfortunately, alternate file streams are not inclued when we download a file normally, else we could download the file on a windows machine and use built in PowerShell or CMD commands to read the alternate stream.

I found an [article on SuperUser](https://superuser.com/questions/1520250/read-alternate-data-streams-over-smb-with-linux){:target="blank"} describing how to list and download alternate data streams over SMB ([9]). I logged into the User share with `smbclient` as usual, then I went into the directory with the "empty" file in question. I then issued an 'allinfo' command on the file in question.

{:refdef: style="text-align: center;"}
![Finding alternate file streams](../../assets/img/writeups/htb/nest/allinfo_commands.png)
{: refdef}

So the name of the alternate stream is Password. Go figure.

We then downloaded the data in the alternate stream with:

`get "Debug Mode Password.txt:Password:$DATA"`

Reading that file gives us a password of **WBQ201953D8w**.

With that, I headed back to the HQK Reporting Service and went into debug mode.

#### HQK Reporting Service Debug Mode

Using `telnet` again I connected to the service and entered debug mode. I listed out the commands we could access.

{:refdef: style="text-align: center;"}
![Debug Commands](../../assets/img/writeups/htb/nest/debug_commands.png)
{: refdef}

I saw 3 new commands we could access:

1. `Session` which listed connection details for the service (not useful)
2. `Service` which listed details about the service (not useful)
3. `ShowQuery <Query ID>` which let us read any query file (in our case, it considered all files to be query files)

Since the service considered any file a query file, I could read any file this user (Service_hqk) had access to! I looked around the parent directory of the service for a bit and found some interesting files in the LDAP folder.

{:refdef: style="text-align: center;"}
![LDAP Folder](../../assets/img/writeups/htb/nest/ldap_admin.png)
{: refdef}

Inside Ldap.conf were credentials for an Administrator. Of course, these credentials had the same form as those found for C.Smith so long ago (ie. they were encrypted). I immediately pasted this value into the modified code I used to decrypt C.Smith's password. Sadly, I got a padding error, no matter how many times I repasted it. I was infuriated to be honest. Then I realized (with some help from the forum) that there was another file in the LDAP folder. A file called `HqkLdap.exe`. A file with the same name as one found in Carl's share a while back. 

#### Cracking the Administrator Password

I took the exe from Carl's folder and sent it to my native Windows machine since Linux told me the exe was a Windows executable. Since it looked to have been writeen in C#, I downloaded a tool called [ILSpy](https://github.com/icsharpcode/ILSpy){:target="blank"}, which is a .NET decompiler ([10]). A decompiler attempts to break down an application into its readable source code. It's not perfect, and does not work for every .NET application, but it did the trick for us in this situation. We opened the exe in the program and searched through the source. I found a file that dealt with encryption/decryption called `HqkLdap.CR`. 

{:refdef: style="text-align: center;"}
![Updated encryption algorithm](../../assets/img/writeups/htb/nest/updated_encryption_algo.png)
{: refdef}

The file had a similar form to the VB script used earlier. It looks like the only difference was the salt, number of iterations, passphrase, and init vector.

{:refdef: style="text-align: center;"}
![Visual Basic code for comparison](../../assets/img/writeups/htb/nest/vb_code.png)
{: refdef}

I had a feeling the actual algorithm was the same, just different values. I simply took the values from the exe, and updated the VB script accordingly. I ran the updated script with the Administrator encrypted password, and got a plaintext password of **XtH4nkS4Pl4y1nGX** back.

**Note: You could also use .NET fiddle to compile the decompiled code as C#, and get the same result**

#### Getting a Reverse Shell With Psexec

Now that I had the Administrator password, I logged into the box with Psexec. From there, I grabbed the root hash, and rooted the box.

{:refdef: style="text-align: center;"}
![Psexec](../../assets/img/writeups/htb/nest/psexec_admin.png)
{: refdef}

### Conclusion

#### Fixes

1. Plaintext credentials for TempUser in a welcome email  
**Fix: Do not allow anonymous SMB login, or update a new user's password the first time they login, so the initial password given is not valid**  
2. Decrypted C.Smith's password by using RU_Scanner script  
**Fix: Update permissions on Carl\ folder to block all Users except C.Smith**
3. Read HQK Debug Password out of an Alternate Data Stream  
**Fix: Do not store any form of credentials in an ADS that is accessbile to a user who should not be able to view it**
4. Decrypted the admin password using RU_Scanner script with updates to certain values  
**Fix: Store the exe which performs the encryption inside a protected folder, only viewable by Administrators for example**

#### Final Thoughts

This has been my longest writeup to date by far. Yet, this was also one of the most satisfying boxes I have done. As I mentioned at the top of this writeup, this box was almost pure enumeration. No well known exploits, just information gathering, windows filesystem tricks, and some interesting code mangling. Thank you to `@VbScrub` for the box creation and `@TazWake` for some helpful nudges. On to the next one...

*As always, if you have any questions or feedback, please contact me on HTB! I am always looking to improve my writing to make it as clear and concise as possible (while remaining somewhat beginner friendly). Happy Hacking!*

### References

1. Samba Wiki on mounting drives locally: https://wiki.samba.org/index.php/Mounting_samba_shares_from_a_unix_client
2. Varonis article on Share vs Local file/foler permissions: https://www.varonis.com/blog/ntfs-permissions-vs-share/
3. Windows documentation on icacls: https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/icacls
4. .Net Fiddle: https://dotnetfiddle.net/
5. MDSec HQK Vulnerability: https://www.mdsec.co.uk/2020/02/cve-2020-0618-rce-in-sql-server-reporting-services-ssrs/
6. Microsoft SQL Server Reporting info: https://docs.microsoft.com/en-us/sql/reporting-services/create-deploy-and-manage-mobile-and-paginated-reports?view=sql-server-ver15
7. Streams software: https://docs.microsoft.com/en-us/sysinternals/downloads/streams
8. What are ADS by Malwarebytes: https://blog.malwarebytes.com/101/2015/07/introduction-to-alternate-data-streams/
9. Listing Alternate Streams in SMB: https://superuser.com/questions/1520250/read-alternate-data-streams-over-smb-with-linux
10. ILSpy github: https://github.com/icsharpcode/ILSpy