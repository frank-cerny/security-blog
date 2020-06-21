---
layout: post
title:  "Road to the OSCP Part 2"
categories: r2oscp weekly-recap
permalink: /:categories/:title
---

## Road to the OSCP Weekly Recap #2

Date: 6/21/2020   
Author: n0tAc0p  

### Discliamer
**This post may contain spoilers for the Hack the Box boxes mentioned. While they are all retired boxes, it still may ruin the experience for some people. Read on at your own risk**

See weekly post #1 for more information on the goal of these posts.

{:refdef: style="text-align: center;"}
![Weekly Progress #2](../../assets/img/r2oscp/weekly/week_2_progress.png)
{: refdef}

### Devel [Windows]

I ended up having to use Metasploit to exploit ms13-053. I tried numerous manual exploits, but I could not for the life of me get anything to work. Oh whale. I don't enjoy using Metasploit, but it was still good practice.

#### Lessons Learned
- Metasploit is a nice lifeline (for 1 machine) if needed
- Differences between asp.net and asp
- All about windows access tokens (not needed for anything, but nice information regardless)

### Bashed [Linux]

Interesting box. It was fun to use a tool developed by the author, and the method to root was fairly straightforward.

#### Lessons Learned
- How to run scrips as other people with sudo -u
- Difference between sudo and su
- How cron and Anacron work, their syntax
- How to find files modified in some time range (ie. last 24 hours)

### Optimum [Windows]

Very well made box. Openened my eyes to many things I never knew about the Windows OS.

#### Lessions Learned
- Different locations x32 and x64-bit binaries are stored on windows
- How to use and compile [Seatbelt](https://github.com/GhostPack/Seatbelt){:target=blank} and [Watson](https://github.com/rasta-mouse/Watson){:target=blank}
- Asp(x) reverse shells are a thing
- How to find the .NET version of a machine
- Basic PowerShell fundamentals

### Nibbles [Linux]

This box was mostly straightforward, except for the need to guess the admin password to gain access to the admin panel. 

#### Lessons Learned
- When bruteforcing is not an option, use common passwords (and box names)
- How to use Hydra (syntax is wack)
- If we control any part of the PATH where a binary is used by `root`, we can most likely become root

### Conclusion

I am pretty happy with how the previous week turned out. I got done 1 more box than I expected too, and only needed minimal hints from IPPSec and google to finish them. My most important takeaway was being able to compile Windows enumeration tools (such as Watson) on my local machine, and use it on a remote machine. Before this week, my brain struggled to do that. Hopefully I can practice such skills in the upcoming week as well!

### Looking Ahead

I hope to complete another 4 retired boxes next week: Jeeves, Bastard, Beep, and Granny. Should be another busy week, so we shall see if that works out. Until next week. Happy hacking!


