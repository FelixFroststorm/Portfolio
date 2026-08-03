#THM #AD 

**OS:** Windows  
**Date:** 28th of July 2026

## Executive summary
In this scenario, we are playing the role of a pentester that will validate a hardened environment after an initial pentesting engagement. Through several steps, we are still able to get credentials to one user, which in turn was used to obtain another user configured with dangerous privileges that was used to escalete privileges to the administrator for the whole domain.  
The initial user had pre-authentication disabled, which means that we were able to find his password through an asreproasting attack. With credentials, we were able to find another set of credentials from a user that was configured with dangerous privileges which in turn gave us administrative privileges, compromising the whole domain. Remediations are suggested at the end of this document.


## Initial setup
First, we connect over VPN where the target is located. 

We then create our folder and start it:
`mkdir THMFusionCorp && cd THMFusionCorp`
In each terminal we are using, we also set the environment variable for ip-address. 
This makes it easier to fetch commands from our knowledge vault and also reduced typos:
`export target=10.112.148.142`

We then ping the target to ensure we are able to reach it:
`ping $target`




## Enumeration
Our initial scans yields these open ports at the target host:

![233](Images/Pasted%20image%2020260728114829.png)

We see several services running on port:
- [ ] 53: DNS
- [ ] 80: HTTP web server
- [ ] 88: Kerberos (Authentication protocol for AD-environments)
- [ ] 135: Microsoft RPC
- [ ] 139: NetBIOS-ssn (used for smb file sharing, technically obsolete)
- [ ] 389: LDAP (Lightweight Directory Access Protocol for directory querying.)
- [ ] 445: SMB (Server Message block for file sharing in AD)
- [ ] 464: Kerberos password change/setup service
- [ ] 593: RPC over HTTP
- [ ] 636 and 3268+3269: LDAPS (encrypted LDAP) and global catalog/global catalog secure
- [ ] 3389: RDP (Remote Desktop Protocol) 
- [ ] 5985: Winrm

Since there is a web server, we start out there, finding four employees, which we note.

![357](Images/Pasted%20image%2020260728114850.png)


We start with checking ldap, rpc and smb for anon/guest access. SMB gives us the domain, which we save in /etc/hosts:

![625](Images/Pasted%20image%2020260728114907.png)


Anonymous/guest user access does not yield anything else. We are able to connect to the RPC service anonymously, but we are unable to list usernames:

![499](Images/Pasted%20image%2020260728114922.png)


Since we want usernames and have 4 likely names, we use a script to generate known versions of usernames:

![](Images/Pasted%20image%2020260728114943.png)

and attempt to use Kerbrute, which does not find a valid username:

![409](Images/Pasted%20image%2020260728114956.png)

Our initial Nmap scap showed a possible backup directory worth checking out:

![327](Images/Pasted%20image%2020260728115009.png)

And sure enough, there is a sheet that gives us a possible list of usernames, which we save in the users.txt file as well:

![259](Images/Pasted%20image%2020260728115024.png)

We attempt to use this new list with Kerbrute, which yields a valid user kparker:

![497](Images/Pasted%20image%2020260728115037.png)

With a valid username, we can attempt asreproasting:
![](Images/Pasted%20image%2020260728115052.png)

We obtain an asrephash which we save into asrephashes.txt that we can attempt to crack for credentials to the lparker user:

![](Images/Pasted%20image%2020260728115103.png)

Since we now have a valid credentialset for lparker, we can attempt to enumerate against services.
We start with attempting kerberoasting, which yields no results:

![](Images/Pasted%20image%2020260728115115.png)

We should attempt credentials towards different services, which works against ldap:
![](Images/Pasted%20image%2020260728115124.png)

We also see that user jmurphy has possible credentials in the description, which we should also check out, but before that we check other services:

![](Images/Pasted%20image%2020260728115140.png)

We are likely to get a shell with winrm but before attempting that we want to list what shares this user has access to:

![](Images/Pasted%20image%2020260728115150.png)

Not very exciting, so we continue with our newfound user jmurphy. Kerberoasting found no entries with this user, so we attempt to authenticate against services. We find that this user has write permissions to C$ and also read permission to the admin share:

![](Images/Pasted%20image%2020260728115201.png)


We also check ldap, smb and rdp and both ldap and winrm vulnerable.

In case we need it, we also set up Bloodhound in case we want to check the environment:

![](Images/Pasted%20image%2020260728115212.png)

The files is then ingested into a Bloodhound instance running on localhost, ready for queries should we need them. 


## Exploitation

Since we saw that we could spawn a shell with winrm with the lparker user, we attempt that to see if we can find something interesting:

![357](Images/Pasted%20image%2020260728115223.png)

We find the flag and then we type it:

![](Images/Pasted%20image%2020260728115235.png)

We are also able to spawn a winrm for user jmurphy:

![](Images/Pasted%20image%2020260728115245.png)

We first find the flag:

![](Images/Pasted%20image%2020260728115257.png)
  
  We also check which rights this user has and we see that the user has two privileges which together likely can be used to gain privilege escalation:
  
![506](Images/Pasted%20image%2020260728115310.png)

**Vulnerability:** These setting likely lets this user download a copy of the sam file (users, password hashes and groups on the local computer) and the system.hive file (contains the cryptographic lock to that file). 


## Post-Exploitation (Privilege Escalation)

We create a temp folder, go into it, ask for the sam file and the system.hive file and then we download it to our attacker machine:

![435](Images/Pasted%20image%2020260728115322.png)

We also get the ntds.dit file, since that contains the usernames, hashes and groups for the whole domain, not just locally. To copy that file we have to do some more setups first.
On the attacker machine:
gedit viper.dsh
Put this into it:
```plain-text
set context persistent nowriters
add volume c: alias viper
create
expose %viper% x:
```
And then run:
unix2dos viper.dsh to convert it to the correct format.
Then we upload it to the victim with our shell and run it with the diskshadow command:

![](Images/Pasted%20image%2020260728115336.png)
We then use robocopy to download the exposed ntds.dit file, which we can then upload to your attacker machine.

![540](Images/Pasted%20image%2020260728115347.png)

Our attacker can then use secretsdump to extract all the usernames and hashes:

![513](Images/Pasted%20image%2020260728115359.png)

All we have to do now is to test for pass the hash attack with the domain controller administrator:

![](Images/Pasted%20image%2020260728115415.png)

We spawn a shell with winrm:

![](Images/Pasted%20image%2020260728115425.png)

And fetch the flag:

![459](Images/Pasted%20image%2020260728115436.png)



## Findings and Flags
In this section, finding from various sources gets collected, typically usernames, hashes, credentials or similar.

Flag for user lparker: THM{c105b6fb249741b89432fada8218f4ef}
Flag for user jmurphy: THM{b4aee2db2901514e28db4242e047612e}
Flag for admin DC: THM{f72988e57bfc1deeebf2115e10464d15}
#### Usernames:Passwords
lparker:!!abbylvzsvs2k6!
jmurphy:u8WC3!kLsgw=#bRY

#### Usernames:Hashes
Local-Admin:
Administrator: :2182eed0101516d0a206b98c579565e6
DC-Admin:
Administrator: :9653b02d945329c7270525c4c2a69c67


## Suggestions / Remediation

### Kerberos pre-authentication should be enabled for all users
**Vulnerability:** Asreproasting due to Kerberos pre-authentication being disabled for a user. This is enabled by default, so at some point this has been changed.
Action: Enable Kerberos pre-authentication for this users, and check for others.

**Remediation steps**
Find vulnerable users in Powershell:

```
Get-ADUser -Filter * -Properties DoesNotRequirePreAuth |
    Where-Object { $_.DoesNotRequirePreAuth -eq $true } |
    Select-Object Name, SamAccountName
```

and remediate (change to other users if others are present):

```
Set-ADAccountControl -Identity lparker -DoesNotRequirePreAuth $false
```

### Credential federation should be changed to a more robust process
**Vulnerability:** User had credentials in plaintext in their user description in ldap. 
**Action:** Credential federation should never have steps that saved a password in plaintext in a way that is visible to others as this breaks the principle of non-repudiation. Use a process that does not break non-repudiation.


### Remove SeBackupPrivilege and SeRestorePrivilege for users who are not administrators
Vulnerability: User had these two privileges enabled: SeBackupPrivilege
 and SeRestorePrivilege. These two together enables a user access all hashes through reading the sam/ntds.dit/system.hive files
 Action: Remove and audit privileges.

To remove privileges, go group policy object (GPO) go to Computer Configuration-->Policies-->Windows settings-->Security settings-->Local policies-->User rights assignment and check:
- Back up files and directories (SeBackupPrivilege)
- Restore files and directories (SeRestorePrivilege)

These should only belong to admins and backup operators.
To validate users that are inside these groups, in Powershell:
Get-ADGroupMember -Identity "Backup Operators"
Get-ADGroupMember -Identity "Server Operators"

If user is inside this group, remove them. If the user have this privileges directly, remove them.
