---
layout: post
title: O24 CTF 'Ghost In The Wall' Writeup
date: 2024-12-21
categories: [HTB, Machines]
tags: [ctf,writeup,wireshark,pcap,linux,privesc]
toc: true
published: true
---
# Ghost in the wall - Outpost24 CTF challenge writeup
## Challenge Description

`What is this?! How did someone manage to capture my network traffic? Must've been that damnable ghost in my walls...`  
  

Find all 3 flags and combine them as such `flag1---flag2---flag3` in order to scare away the ghost in your walls for good!  
**Note:** You are sharing this challenge with the other competitors. Be sneaky, lest you want other teams to know your solution!

## Solution
Along with the challenge description you get a pcap with ~10k frames, containing a bunch of encrypted QUIC traffic, as well as TLS encrypted TCP, and some unencrypted HTTP over TCP to sites dedicated to cars.

### PCAP Analysis
First, let's just do a string search for 'Flag' to see if there's an easy win.  
Indeed, the first part of the flag is in a parameter to a HTTP GET request.
![pcap_flag_1](https://tuxpad.github.io/assets/images/ctf/o24/ghost_01.png)  

Next, in order to get an overview of the type of traffic in the pcap,we'll select `Statistics -> Protocol Hierarchy` from the toolbar menu.
This gives us an overview of what we have to work with, like how 74.9% of the traffic is TLS.
One thing that sticks out is the seven packets of 'HTML Form URL Encoded'; that might be worth looking at.
![pcap_protocol_hierarchy](https://tuxpad.github.io/assets/images/ctf/o24/ghost_02.png)  

So, we'll right click on it and apply it as a filter. Immediately, two packets stand out, as they are directed to a different IP address than the rest of the traffic in the pcap.  
![pcap_url_form](https://tuxpad.github.io/assets/images/ctf/o24/ghost_03.png)  

Wait a sec, this is a reverse shell! The author of the challenge isn't going to attack some random site, so this is a clear indicator that we should have a closer look.
![pcap_reverse_shell](https://tuxpad.github.io/assets/images/ctf/o24/ghost_04.png)  

We follow the stream, which provides more useful information.  
In addition to the reverse shell attempt, we can see that the server responded 'b64 command sent', and a second POST request with a hint that it din't work.  
![pcap_stream](https://tuxpad.github.io/assets/images/ctf/o24/ghost_05.png)  

### Getting a foothold
When we have a look in the browser we get to a login screen that we'll need to bypass.  
![login_portal](https://tuxpad.github.io/assets/images/ctf/o24/ghost_06.png)  

We can try something like SQLi, but that doesn't work. However, in the pcap we could also see that the POST request also had a session cookie set, and when applying it we bypass the login.  
![login_bypass](https://tuxpad.github.io/assets/images/ctf/o24/ghost_07.png)  

Now, the server response said base64 encoded, but the shell was sent in clear text, and then we saw that it didn't work. So, we'll just craft our own reverse shell command, base64 encode it and give it a shot, and we get a shell!  

### Getting user account
In the base directory we can se the document `routes.py`, which contains the backend for the login, made in Flask. Here we can see the hard coded credentials for the login. A password is always good to collect, as it might be reused somewhere (though that's not the case in this challenge).  
![linux_flask_app](https://tuxpad.github.io/assets/images/ctf/o24/ghost_08.png)  

Now we can start doing some local host enumeration; check OS version, environment variables, user accounts, groups, etc.  

Inside `/etc/group` we find something interesting; it looks like a password for the only user account.  
![linux_groups](https://tuxpad.github.io/assets/images/ctf/o24/ghost_09.png)  

Indeed it is, and we can collect the second part of the flag from `~/user.txt`  
![flag_2](https://tuxpad.github.io/assets/images/ctf/o24/ghost_10.png)  

### Root privesc
Since we have the users password, and we're not worried about being detected on the system, let's check if the user has sudo privileges.  
It turns out that we may run `sudo /snap/bin/ls -la /root`
![sudo_l](https://tuxpad.github.io/assets/images/ctf/o24/ghost_11.png)  

So let's do just that! We can see the file `/root/root.txt`, which obviously is where we'll find the final part of the flag.  
![root_contents](https://tuxpad.github.io/assets/images/ctf/o24/ghost_12.png)  

When you find that you have access to a particular command as sudo, you should always consider how that can be abused, particularly if that's the only command you can run as a super user. Perhaps you can pipe something to the command, get the program to run in interactive mode or you can escape it to a shell.  
  
In this case, we can see that although the program (`/snap/bin/ls`) is owned by root, and we only have permission to execute and read, the directory `/snap/bin` has write permission for all set. This means that we can't manipulate the contents of the file, we can do things like rename, move, or delete it! This means that we can replace it with a shell script, which we then can run in sudo.  
![rename_ls](https://tuxpad.github.io/assets/images/ctf/o24/ghost_13.png)  


We'll just rename ls,
![rename_ls](https://tuxpad.github.io/assets/images/ctf/o24/ghost_14.png)  

creeate a shell script to cat the flag (we could get a shell, but there's no need),
[shell_script](https://tuxpad.github.io/assets/images/ctf/o24/ghost_15.png)  
  
chmod it to be executable, and then run it to get the final part of the flag.
[flag_3](https://tuxpad.github.io/assets/images/ctf/o24/ghost_16.png)  
