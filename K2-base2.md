---
Status: "#Complete"
OS:
ip:
Start_Time: 2026-02-22 12:08
---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning
	I used nmap to scan for details of the target. Following the result, I know that this is an active directory.
```bash
# Nmap Full Scan
nmap -p- --min-rate 1000 -oN nmap_full.txt <IP>
```
![](<assets/Pasted image 20260222121655.png>)
	Now let's do an active directory enumeration.
	I check the smb signing and it needs signing.  Also I check for null session, the target doesn't allow null session.
	So I used kerbrute to enumerate the possible usernames and get one valid.
![](<assets/Pasted image 20260222123554.png>)
	The task told us to use the information that from the base camp, so I choose to use the full names of the two users to generate a list for kerbrute.

![](<assets/Pasted image 20260222154508.png>)
	Next, I ran the kerbrute and get two results.

![](<assets/Pasted image 20260222154528.png>)
	After getting the valid credential, I tried to use the same password to login. And I found that rose is using the same password.

![](<assets/Pasted image 20260222160424.png>)
	
---

##  3. Initial access
	Next, I use the credential of rose to login.

```
evil-winrm -i 10.80.144.107 -u r.bud -p 'vRMkaVgdfxhW!8'
# Or you can use impacket as a substitute
impacket-wmiexec k2.thm/r.bud:'vRMkaVgdfxhW!8'@10.81.145.106
```
	This gives me a powershell on target. And I found two valuable information on target's desktop.These are about the password and password changing policy.

![](<assets/Pasted image 20260223120944.png>)

![](<assets/Pasted image 20260222161051.png>)
	And I upload the sharphound.ps1 and then using bloodhound to gather more information about privilege escalation path. I upload the Sharphound.exe first and run it.

![](<assets/Pasted image 20260222161128.png>)
	![](<assets/Pasted image 20260223122406.png>)
	Next， I download the zip file and analyze it with bloodhound.

![](<assets/Pasted image 20260223133315.png>)
	Bloodhound tells us that I can escalated directly to administrators.
	Once I escalated to k2server.k2.thm, I will be able to get the admin's hash because k2server has DCSync  privlege. Also I found that james is member of IT staff, and by the hint from the files in r.bud's document, I now have a more clearly escalation path.
![](<assets/Pasted image 20260223165026.png>)
	
	


##  4. Privilege Escalation

### 4.1 Local Enumeration
*Record findings from automated scripts and manual environment checks.*
- [ ] **Automated Tool:** Run [[PEASS-ng]] (winPEAS.exe / linpeas.sh).
- [ ] **Kernel Version:** `systeminfo` (Windows) or `uname -a` (Linux).
- [ ] **Sensitive Files:** Check for `.txt`, `.pdf`, `.zip` in user folders or web roots.
- [ ] **Misconfigurations:** (e.g., SUID, Sudo -l, Unquoted Service Paths, Token Impersonation).
	According to the two notes. I used a script to create a passwords list and brute force it.

```
> passwords.txt

for d in $(seq 0 9); do
  for s in '!' '@' '#' '$' '%' '^' '&' '*' '.' '-' '_' '+' '=' ':'; do
    echo "${d}${s}rockyou" >> passwords.txt
    echo "${s}${d}rockyou" >> passwords.txt
  done
done

wc -l passwords.txt
```
```
crackmapexec smb ip -d k2.thm -u j.bold -p passwords.txt
```

	And I get the credential.

![](<assets/Pasted image 20260223172424.png>)
	With this credential, I can try to use impacket to force to change j.smith's passwords so that I will have access.

```
impacket-changepasswd -altuser j.bold -altpass '#8rockyou' -newpass 'nm416414' -reset k2.thm/j.smith@10.80.131.217
```
	I force it to change the password.

![](<assets/Pasted image 20260223173646.png>)
	Success. Now I can login as j.smith with eveil-winrm.
### 4.2 Escalation Path
	Now after I login as J.smith, first I check the privlege it has.
	
![](<assets/Pasted image 20260223180346.png>)
	This means I can get the backup through this privlege.
	And then I check the C:\Windows\NTDS directory and see the files it has. Also I check the SYSVOL directory.
	It has ntds.dit file so it is highly possible a DC(domain controller). This means I can use the seBackup privlege to view the hashes of other users.

![](<assets/Pasted image 20260223180926.png>)

![](<assets/Pasted image 20260223181018.png>)
	First, I ctry to use the reg to get the system.bak and sam.bak files.

```
reg save HKLM\SYSTEM C:\Users\j.smith\Desktop\SYSTEM.bak /y
reg save HKLM\SAM C:\Users\j.smith\Desktop\SAM.bak /y
```

![](<assets/Pasted image 20260223185431.png>)
	Next, I download it and use impacket-secretdump to crack it.
	And I get the admin's ntlm hash.

![](<assets/Pasted image 20260223185517.png>)
	Following the result, I can use the hash to login as admin.
![](<assets/Pasted image 20260223185550.png>)
	And by listing the directories in users I can know the users that are in this DC.

![](<assets/Pasted image 20260223185816.png>)
	

	

	
	

---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type    | Flag Content (Hash)                   | Screenshot (Link)                           |
| :----------- | :------------------------------------ | :------------------------------------------ |
| **user.txt** | THM{3e5a19a9ba91881f4d7852d92126a97f} | ![](<assets/Pasted image 20260223173928.png>) |
| **root.txt** | THM{a7e9c8149fec53865eff983143b1f5ba} | ![](<assets/Pasted image 20260223185626.png>) |

---

