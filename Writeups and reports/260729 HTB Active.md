#HTB #AD 

**OS:** Windows  


## Executive summary

On this active directory machine, we are able to get credentials from a user because there is a public share open to anyone through anonymous access. The username was listed together with the password which was encrypted with AES. Microsoft stores the decryption key in their documentation, which makes it unsecure and easily crackable. With valid credentials, at Kerberoasting attack was conducted and we were able to get the Administrator Kerberos hash. This hash was crackable as well, which yielded that Administrator password, which fully compromised the domain. Suggestions for remediation is located at the end of the document.

## Initial setup
First, we connect over VPN where the target is located. 

We then create our folder and start it:
`mkdir HTBActive && cd HTBActive`
In each terminal we are using, we also set the environment variable for ip-address. This makes it easier to fetch commands from our knowledge vault and also reduced typos:
`export target=10.129.48.202`

We then ping the target to ensure we are able to reach it:
`ping $target`


## Enumeration
Our initial scans yields these open ports at the target host:
![294](Images/Pasted%20image%2020260729081148.png)


We first attempt anonymouse and guest access to ldap, smb and rpc. The guest account seems to be disabled, but we are able to list shares with anonymous:

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

We attempt to get the --sam file from smb, which we do:

![](Images/Pasted%20image%2020260729093419.png)

After that little sidetrack, we spawn a shell as the administrator:

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
Several parts made the compromise possible. Here are suggested remediations:

#### Deny anonymous access
The file with credentials was accessed due to anonymous access being open. If shares has to have anonymous access, ensure that no sensitive information is left on it.

#### Credentials should not be saved in files
The initial access was obtained due to an xml-file that included both username and an encrypted password that was crackable. The specific variable that stores the password was the cpassword attribute. Microsoft has released a securitybulletin to remediate the issue which can be read here: 
https://learn.microsoft.com/en-us/security-updates/securitybulletins/2014/ms14-025

#### Administrative account should never be configured as a service account
The administrator account was configured with a SPN, making it a service account that should never have administrative rights. In this environment, the administrative Kerberos hash was obtained because a valid user with valid credentials asked the KDC for a ticket granting ticket, then asked the KDC for a service ticket which was then cracked offline to obtain the password of the administrator. The Administrator also had a short password, which means that it was easily crackable.

#### Consider establishing or updating the password policy
Administrators should never have accounts with easily crackable passwords. Administrators (at least) should not have passwords that are easily crackable, and increasing the password length will greatly defend against many attacks. Establish a policy of at least 24 characters.