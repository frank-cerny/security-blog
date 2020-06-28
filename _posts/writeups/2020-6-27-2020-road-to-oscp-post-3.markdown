---
layout: post
title:  "Road to the OSCP Part 3"
categories: r2oscp weekly-recap
permalink: /:categories/:title
---

## Road to the OSCP Weekly Recap #3

Date: 6/27/2020   
Author: n0tAc0p  

### Discliamer
**This post may contain spoilers for the Hack the Box boxes mentioned. While they are all retired boxes, it still may ruin the experience for some people. Read on at your own risk**

See weekly post #1 for more information on the goal of these posts. 4 Windows boxes and a single Linux box this week. Learned a BUNCH about windows Kernel exploits. I also dislike Windows Server 2003 R2. That is all.

{:refdef: style="text-align: center;"}
![Weekly Progress #3](../../assets/img/r2oscp/weekly/week_3_progress.png)
{: refdef}

### Jeeves [Windows]

This box was a major lesson in patience. I required help to find the Jenkins endpoint needed for the foothold exploit. All because I did not let my gobuster session finish. Figures.

### Major Hangups
- Not letting my enumeration scripts finish becasue they were running slowly

#### Lessons Learned
- Be patient when enumerating. Stick to the plan and make sure everything finishes before freaking out
- How to leverage Nishang reverse shells
- How to get a Metasploit session from a standard PowerShell reverse shell
- Full anonymous Jenkins access = RCE
- Rotten Potato and Juicy Potato are useful exploits if a user has the SeImpersonate Privilege (common on Service accounts)

### Beep [Linux]

Holy services. Another lesson in patience. Still did not let my gobuster finish because it was taking too long.

#### Major Hangups
- Not letting my enumeration scripts finish (AGAIN)
- If an exploit is not working that we KNOW should work, try a few times or reset the box and try again

#### Lessons Learned
- Elastix and Asterisk are a suite of VOIP tools, packaged into a single appliance
- Shell Shock is more common than I expected
- What Local File Inclusion was, and how to possibly turn it into Remote File Inclusion (and RCE)
- If an exploit is not working, and not printing any error messages, try to proxy it through Burp and debug network responses (Burp Free is amazing)

### Bastard [Windows]

A 3rd (and hopefully final) lesson in patience. The only hiccup I had was finding the REST endpoint path needed for the Drupal exploit. Can you guess why?

### Major Hangups
- Did.Not.Let.Enumeration.Finish

#### Lessions Learned
- Learned a few different ways to bypass PowerShell code execution restrictions
- droopescan is a thing, useful for enumerating open source CMS's
- Firewalls will generally prevent executing things from network shares (not in HTB labs, but in real life)

### Granny & Grandpa [Windows]

I honestly did not enjoy these much. The boxes were unstable, and they were *nearly* carbon copies of each other.

#### Major Hangups
- Did not know what WebDav was and why it could be useful

#### Lessons Learned
- If we can't get any normal shell, use meterpreter
- Windows Server 2003 R2 is wonky, and quite unstable
- WebDav is extremely useful, and at times can lead to RCE
- Tools to enumerate WebDav include davtest, cadaver, and curl

### Conclusion

I was happy to complete 5 boxes this week, but somewhat disappointed at the amount of help required. 80% of the help I needed was because I did not let my enumeration scripts complete. After failing 3 times because of it, I hope I have learned my lesson. Still, quite a lot of good lessons this week, even if 4/5 boxes required Kernel exploits to get System (funnnnnnnnnnnnnnnnnnnnnnnnnn).

### Looking Ahead

I plan on taking the next week off. I want to focus on beefing up my automated enumeration script, update how I take notes, and even move my Playbooks to GitBook or something similar. If I happen to have some extra time, I want to attempt one of the more challenging boxes. I will still write an update post, and try to explain some of the changes I made and why. Have a great week, and happy hacking!


