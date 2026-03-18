# Year of the Dog

## Overview
- Platform: TryHackMe
- Category: Web Exploitation / Linux Privilege Escalation
- Difficulty: Hard

## Key Techniques
- Cookie parameter SQL injection with WAF bypass
- Web shell write and reverse shell execution
- Gitea abuse (basic auth, 2FA bypass, webhook execution)
- Container-to-host pivot and SUID-based root escalation

---
Title: "Year of the dog - TryHackMe Writeup"
Author: Cankun Wang
date: 2025-12-10
tags: [tryhackme, writeup]
---

# Task
![image-20251210163404417](./assets/image-20251210163404417.png)

# Enumeration
We start with a normal nmap scan for all ports.

![image-20251210163447882](./assets/image-20251210163447882.png)

Nothing interesting find. Only port 80 and 22.

Let's view the target website and try to use gobuster for the directories.

![image-20251210163555425](./assets/image-20251210163555425.png)

We noticed that there is a sentence called you are number 31 in the queue. We are not sure about what the true meaning it is. May be means admin will view something? Let's proceed to gobuster first.

![image-20251210165733448](./assets/image-20251210165733448.png)

We find two potential directories here.

![image-20251210170648913](./assets/image-20251210170648913.png)

dirsearch result is almost same with gobuster. Let's check these directories.

![image-20251210170753049](./assets/image-20251210170753049.png)

This is a useful place. It may contain several vulnerabilities.

We will check these directories.

![image-20251210171033816](./assets/image-20251210171033816.png)

We find some otf files.

![image-20251210171049447](./assets/image-20251210171049447.png)

![image-20251210171107187](./assets/image-20251210171107187.png)

![image-20251210171115313](./assets/image-20251210171115313.png)

And the directory of css listed some info, points toward the otf files. This make sure that the front's structure. However, I tried dot dot slash and it is not working. There are not enough info or possible path leaves for us. We may need to make some changes.

I noticed that the website's main page is showing " You are number ** in queue". Let's intercept the request and have a see.

![image-20251222131248472](./assets/image-20251222131248472.png)

Here, Cookie has a parameter "id=*****". This is a potential vulnerability of sql injection. 

---

$id = $_COOKIE['id'];
$sql = "SELECT * FROM users WHERE id = '$id'";

---

This is the possible back end query sentence.

Let's try this.

![image-20251222132549617](./assets/image-20251222132549617.png)

We have error return, which means sql injection here is working. We find a potential path.

![image-20251222132644583](./assets/image-20251222132644583.png)

When we try the boolean bypass, no error returned, which means we may successfully bypassed. We need a further verification.

![image-20251222132817692](./assets/image-20251222132817692.png)

However, when we tried the sleep bypass, we are intercepted. This means the WAF may have a simple word match dictionary. 

Next step, let's try use union select and set the parameter to null.

![image-20251222133251820](./assets/image-20251222133251820.png)

We didn't get what we want. For now, we have make sure that UNION is available since WAF doesn't block it. And select null,null doesn't return error message. Also, we make sure that website has no path for file upload or any potential vulnerabilities in front end. Now, we can try the sql load file function according to all these infos. 

  ![image-20251222134542159](./assets/image-20251222134542159.png)

We are on the correct way. We can load the file, which means we can try to upload the file and achieve RCE.

The target website has a high possibilities that it is able to run the php. Cookies, apache and mysql are all used in this website, so it is highly possible to run php. 

![image-20251222140041023](./assets/image-20251222140041023.png) 

This is a simple php command and I encode it to hexadecimals to bypass the potential WAF.

![image-20251222140446700](./assets/image-20251222140446700.png)

Let's view this.

![image-20251222140604541](./assets/image-20251222140604541.png)

Now this means we have a cmd here. Let's use this to get the reverse shell.

![image-20251222141522474](./assets/image-20251222141522474.png)

We write a php reverse shell and hold a server to wait for wget from the target.

![image-20251222141557727](./assets/image-20251222141557727.png)

---

![image-20251222152654511](./assets/image-20251222152654511.png)

---

This is the shell we use.

![image-20251222152719944](./assets/image-20251222152719944.png)

After using wget get the file, we trigger it and get the reverse shell.

# Privilege Escalation
![image-20251222152905922](./assets/image-20251222152905922.png)

We have a user named dylan.

![image-20251222152935029](./assets/image-20251222152935029.png)

![image-20251222153008268](./assets/image-20251222153008268.png)

We have a file that show us the name and email of dylan. Let's view other files.

![image-20251222153455650](./assets/image-20251222153455650.png)

We have a work analysis here, but it is a large file. Let's try to grep something out.

![image-20251222153607011](./assets/image-20251222153607011.png)

We find the username, and it looks like it is the password for dylan.

![image-20251222153833573](./assets/image-20251222153833573.png)

We are in. Now we need to find a path to root.

![image-20251223112941443](./assets/image-20251223112941443.png)

After I login as dylan, I checked many regular escalation path. However, these are all not available. ![image-20251223114151906](./assets/image-20251223114151906.png)

I checked the port that is listening. I noticed that there are some abnormal ports.  

port 3000,3306 and 37129 are both internal ports, so we need to use the ssh tunneling.

![image-20251223120834569](./assets/image-20251223120834569.png)

After tunneling, I find that this port may run a web service.

![image-20251223120919407](./assets/image-20251223120919407.png)

Correct. This is a web page.

![image-20251223121016162](./assets/image-20251223121016162.png)

Let's try to login.

![image-20251223121037770](./assets/image-20251223121037770.png)

Hmmm.... We need a passcode. It is protected by 2FA. (You can skip this and scroll down to the next part)

![image-20251223211427911](./assets/image-20251223211427911.png)

![image-20251223211452794](./assets/image-20251223211452794.png)

However, we noticed that this website is powered by Gitea. Let's google it.

We have two findings. One is the Gitea official documentation about API usage, and another is CVE-2021-45331.

On the official documentation, we find that Gitea supports basic http authentication.

![image-20251223212324676](./assets/image-20251223212324676.png)

This is a good point, because we have the username and password. So we could use http basic authentication to login. CVE-2021-45331 verify this. And we can find that the gitea version is 1.13.0, which is lower than 1.5.

![image-20251223212924332](./assets/image-20251223212924332.png)

---

$ curl --url https://yourusername:password@gitea.your.host/api/v1/users/<username>/tokens
[{"name":"test","sha1":"","token_last_eight:"........":},{"name":"dev","sha1":"","token_last_eight":"........"}]

---

![image-20251223213922052](./assets/image-20251223213922052.png)

It indeed accept the basic authentication.

![image-20251223214236934](./assets/image-20251223214236934.png)

Using burp we can intercept our request and modify it.

![image-20251223214353169](./assets/image-20251223214353169.png)

We right click the request and open it in browser, now we can see the content.

![image-20251223214719809](./assets/image-20251223214719809.png)

We find a Test-Repo, and I want to check whether there is a possible webhook. This is a git like platform and it highly possible to use git. 

![image-20251224121005474](./assets/image-20251224121005474.png)

I tried, however, it is intercepted by 2FA. If we want to use PAT, we need to get access the user setting. But the web page we are in does not open the user setting. So we need to think from another way, can we bypass or delete the 2FA?

## Delete 2FA
It is not that possible, we can still check it.

![image-20251224121404623](./assets/image-20251224121404623.png)

This is the db file. Let's try.

![image-20251224121756882](./assets/image-20251224121756882.png)

![image-20251224121807912](./assets/image-20251224121807912.png)

We check the tables.

![image-20251224121852606](./assets/image-20251224121852606.png)

We find the 2FA tables.

![image-20251224122103111](./assets/image-20251224122103111.png)

We just delete it.

![image-20251224122613324](./assets/image-20251224122613324.png)

Now we can simply login and no 2FA exist. To be honest, we can do this after get the shell. And no need for using the basic auth. 

![image-20251224123038728](./assets/image-20251224123038728.png)

Now we can do the git push.

# Privilege Escalation Part 2
So we have the prerequisite of using git hook to escalation privilege.  

![image-20251224124710283](./assets/image-20251224124710283.png)

We go to the setting and create a post receive hook.

![image-20251224124847693](./assets/image-20251224124847693.png)

Next, we trigger it.

![image-20251224135934813](./assets/image-20251224135934813.png)

(I restart my VM, so this is the second time and that's why the listening port is not the same with the above payload)

But we are in. And let's keep checking.

![image-20251224140118212](./assets/image-20251224140118212.png)

We are user git and we find some interesting directories here.

All these are root privilege.

![image-20251224140424121](./assets/image-20251224140424121.png)

And we find that it seems we can run all the commands. Let's use sudo su.

![image-20251224140453445](./assets/image-20251224140453445.png)

We are root now!

However, it seems we are in the docker.

![image-20251224141010848](./assets/image-20251224141010848.png)

The directories seems like the host file system. Let's check.

![image-20251224141203586](./assets/image-20251224141203586.png)

I create a file in /data, because data seems contain the least file numbers.

Let's go back to user dylan to check if it can see the file.

![image-20251224141508480](./assets/image-20251224141508480.png)

We can see the file, which means the host file system is connected with the docker.

![image-20251224142522407](./assets/image-20251224142522407.png)

We first copy the bash into the gitea directory in host side. 

![image-20251224142608410](./assets/image-20251224142608410.png)

So we can find the bash in docker. The next step is give the suid to the bash because we have root privilege here.

![image-20251224142708523](./assets/image-20251224142708523.png)

![image-20251224142731145](./assets/image-20251224142731145.png)

![image-20251224142747900](./assets/image-20251224142747900.png)

Now the bash in host side has the root privilege.

## Key Takeaways
- The main path combined SQL injection, file write, and command execution.
- Gitea and 2FA handling provided a practical pivot for privilege growth.
- Container-to-host relationship validation enabled final root control.

Thanks for reading!



