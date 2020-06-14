---
layout: post
title:  "Road to the OSCP Part 1"
categories: r2oscp weekly-recap
permalink: /:categories/:title
---

## Road to the OSCP Weekly Recap #1

Date: 6/14/2020   
Author: n0tAc0p  

### Discliamer
**This post may contain spoilers for the Hack the Box boxes mentioned. While they are all retired boxes, it still may ruin the experience for some people. Read on at your own risk**

This is the initial post in my Road to the OSCP path. Like so many others on this journey, I decided to keep a blog to keep track of how each week went. The first step in this journey is doing the 40 boxes that `TJnull` suggests. I could not find the actual tweet by him, but the picture can also be found [here](https://www.reddit.com/r/oscp/comments/cu6jhb/updated_oscplike_boxes_from_hackthebox_by_tjnull/){:target="blank"} in a reddit thread mentioning he released it.

Each post will contain an overview of the machines I did during the week, as well as some of the lessons, tools, procedures, etc that I picked up during the week. I will make every effort to not use Metasploit at all. I attempt to root the machine without it, then go back and use Metasploit for the practice of using it. I created an excel doucment to keep track, and each week I will post a picture of the document showing my progress for the week.

{:refdef: style="text-align: center;"}
![Weekly Progress #1](../../assets/img/r2oscp/weekly/week_1_progress.png)
{: refdef}

### Lame [Linux]

The first retired box I have ever done!! This was one of the few boxes I was able to do without getting any form of help (forum, youtube, etc.)

#### Lessons Learned
- Take your time to read the details in your nmap scan. I ran through the results quickly and focused less on the details, and more on the port numbers. My two part exploit path took hours, the "intended way" took minutes
- If distcc is open, you should be able to execute remote commands
- If any SMB program (smbclient, smbmap) has protocol errors, chances are the machine is running SMB 1 (Google how to force those programs to work with SMB 1.0)

### Legacy [Windows]

I spent hours running different versions of the same exploit. Turned out I bricked the box early on, and a reset was all I needed for the exploits to work (RIP).

#### Lessons Learned
- My nmap scan only ever runs default scripts. There are a host of other types of scripts that can be ran on a box (safe, vuln, etc). The nmap engine is much more complex than I anticipated.
- How to use searchsploit to not only search for exploits, but to copy them to a local directory
- Take your time and take breaks. On this box especially, I tried the same exploit for hours, without moving. Coming back the next day and hitting the reset button (literally) was all I needed

### Brainfuc* [Linux]

Wowzers, was this a fun one. If I had not had a complete brain shutdown on the crypto portion, I might have been able to get through it without IPPSec.

#### Lessions Learned
- When dealing with wordpress, use [wpscan](https://wpscan.org/){:target="blank"} and use searchsploit on any of the plugins found. Plugins seem to be the most vulnerable piece of the entire platform
- If we can edit theme files on WordPress, we can gain RCE privileges
- We can create web requests and forms in a file (like one we might find in a searchsploit result) then navigate to it in our browser by hosting it with a python simpleHTTP server
- Use [Boxentriq's Cipher Identifier](https://www.boxentriq.com/code-breaking/cipher-identifier){:target="blank"} to make determing the type of cipher easy (or use a simple known plaintext attack)

### Shocker [Linux]

I got stuck on the enumeration part of this one. I had seen cgi-bin in so many other boxes, I did not even think to use it. It did not help that I have never heard of the shell shock vulnerability either.

#### Lessons Learned
- Shell Shock is a vulnerability that allows bash to execute commands strategically placed after setting an enviornment variable (pretty insane). We could use cgi-bin because we could set enviornment variables through headers (and hence execute code)
- Take version numbers of services and attempt to determine the underlying OS version. It may come in handy later

### Blue [Windows]

I knew before going in this had to do with Eternal Blue. Yet it still took me 2 days to finish. The exploit I was working with formed a connection with my local machine, but crashed immediately. Once I switched over to another exploit, it finally worked.

#### Lessons Learned
- The difference between a staged and stageless metasploit payload
- If one exploit does not work, take a step back and try another one

### Conclusion

For my first week, it was not completely disheartening. I definitely learned a ton, but also made a ton of mistakes. The top 2 things I learned this week were:

1. Take your time; It's not a race. I know the OSCP is timed, but I know I can manage it. The most important thing it to use my time wisely and take breaks (or use another approach) when things are not working "as planned"
2. Google everything. Versions, names, etc. Chances are, someone else has seen it, or done something similar. Google Fu skills are arguably the most important for the OSCP.

### Looking Ahead

Headed into next week, I hope to complete 3 boxes, devel, bashed, and optimum. It's going to be a busy week, otherwise I would attempt 4 or 5 again. Week 1 under my belt, its time for week 2.


