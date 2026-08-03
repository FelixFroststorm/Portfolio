#HTB #AD 

**OS:** Windows  
**Date:** 29th of July 2026

# Executive summary

On this active directory machine, we are able to get credentials from a user because a public share was open to anyone through anonymous access, containing a file with credentials. The username was listed together with the password which was encrypted with AES. Microsoft stores the decryption key in their documentation, which makes it unsecure and easily crackable. With valid credentials, a Kerberoasting attack was conducted and we were able to get the Administrator password because the Administrator was configured as a service account and the administrator also had a relatively short password, which fully compromised the domain. Suggestions for remediation is located at the end of the document.


## Initial setup
First, we connect over VPN where the target is located. 

We then create our folder and start it:
`mkdir HTBActive && cd HTBActive`
In each terminal we are using, we also set the environment variable for ip-address. This makes it easier to fetch commands from our knowledge vault and also reduces typos:
`export target=10.129.48.202`

We then ping the target to ensure we are able to reach it:
`ping $target`


## Enumeration
Our initial scans yields these open ports at the target host:
![294](Images/Pasted%20image%2020260729081148.png)


We first attempt anonymous and guest access to ldap, smb and rpc. The guest account seems to be disabled, but we are able to list shares with anonymous:

![](Images/Pasted%20image%2020260729082149.png)

We note the domain active.htb and then we attempt to connect to smb to read the contents:

![](Images/Pasted%20image%2020260729082715.png)

After looking around inside the share, we find two interesting files, groups.xml that looks like it also contains the password:

![](Images/Pasted%20image%2020260729084643.png)

Before attempting to authenticate, we start a kerbrute scan and also an enum4linux script. It does not yield much interesting.
The string in cpassword is encrypted with AES. We attempt decryption, which works:

![](Images/Pasted%20image%2020260729090530.png)

We now likely have a valid set of credentials which we can use to attempt asreproasting or authenticate to services. We attempt to use it against ldap, smb and rpc and we find that there is a share we can now read, likely containing users:

![](Images/Pasted%20image%2020260729090813.png)

We are able to connect to SMB with these credentials, so we navigate to the desktop of the user in question to fetch the flag:

![](Images/Pasted%20image%2020260729091427.png)

Since we have valid credentials, we can also attempt kerberoasting, which yields the Administrator TGS-REP ticket:

![](Images/Pasted%20image%2020260729091903.png)


## Exploitation

  We attempt to crack it with john the ripper, which gives us the password:

![](Images/Pasted%20image%2020260729092114.png)

We then spawn a shell as the administrator:

![](Images/Pasted%20image%2020260729093454.png)

We go to the desktop to fetch the flag:

![](Images/Pasted%20image%2020260729093727.png)


  


## Findings and Flags
In this section, finding from various sources gets collected, typically usernames, hashes, credentials or similar.

Userflag: 9513e6ca54ffde4c4f75bf289efad21b

domain: active.htb

#### Usernames:Passwords

SVC_TGS:GPPstillStandingStrong2k18
Administrator:Ticketmaster1968


## Suggestions / Remediation
Several parts made the compromise possible. Here are suggested remediations in order of severity.
### Credentials should not be saved in files
**Severity: Critical**
**Technique:** Credential access, T1552.006 (https://attack.mitre.org/techniques/T1552/006/)

**Description:** The initial access was obtained due to an xml-file that included both username and an encrypted password that was crackable. The specific variable that stores the password was the cpassword attribute. 

**Remediation:** Microsoft has released a securitybulletin to remediate the issue which can be read here: https://learn.microsoft.com/en-us/security-updates/securitybulletins/2014/ms14-025 
Also, remove any files in shares that has the cpassword attribute set.
### Administrative account should never be configured as a service account 
**Severity: Critical**
**Technique:** Credential access, T1558.003 (https://attack.mitre.org/techniques/T1558/003/)

**Description:** The administrator account was configured with a SPN, making it a service account that should never have administrative rights. Using valid credentials (SVC_TGS), we obtained a TGT and used it to request a service ticket (TGS) for the Administrator account's SPN. The KDC returned a TGS encrypted with the service account's password hash, which we cracked offline to recover the Administrator password.

**Remediation:** Do not assign SPNs to privileged accounts. Service accounts should follow least privilege and must not be members of privileged groups such as Domain Admins. 

### Establish or update the password policy
**Severity: High**
**Technique:** Credential access, T1110.002 (https://attack.mitre.org/techniques/T1110/002/)

**Description:** The Administrator also had a short password, which means that it was easily crackable. Administrators (at least) should not have passwords that are easily crackable, and increasing the password length will greatly defend against many attacks. Establish a policy of at least 24 characters for Administrative accounts.

**Remediation:** Establish or update the password policy in regards to length for Administrative users.
### Deny anonymous access
**Severity: Medium**
**Technique:** Discovery, T1135 (https://attack.mitre.org/techniques/T1135/)
**Description:** The file with credentials was accessed due to anonymous access being open. If shares have to have anonymous access, ensure that no sensitive information is left on it.

**Remediation:** Restrict anonymous/null-session access to shares.