#THM #AD

**OS:** Windows  
**Date:** 25th of July 2026
## Executive summary
In this Windows environment, we were able to get administrative privileges by exploiting a vulnerability in the certificate template configuration. The vulnerability enables a user to request a valid certificate from anyone in the group "authenticated user". As long as we have credentials, we are able to request a valid certificate, which we can then use to authenticate to any service as an administrator. To get the user access, we were able to list all users with anonymous access through ldap and in the description of two users, the password credential was shown in cleartext which is likely a default password set by an administrator. In this environment, there is several suggested remediations:
- Disable anonymous access (Used to get initial foothold)
- Disable guest accounts
- Remediate specific certificate vulnerability
- Change password distribution methods

A structured approach to several of the issues should be in place, and a gap analysis between current state and desired state should considered and implemented to reduce the likelihood of this happening going forward.  

## Initial setup
First, we connect over VPN where the target is located. We also create a folder representing the machine we are targeting, go into it and save the ip-address as the variable in our terminal before starting. 

Creating our folder and starting off inside it:
`mkdir THMLedger && cd THMLedger`
The purpose of this is that all logs, payloads and collected information stays structured inside a folder related to the project.

In each terminal we are using, we also set the environment variable for ip-address. This makes it easier to fetch commands from our knowledge vault and also reduced typos:
`export target=10.113.150.73`

## Enumeration
Our initial scans yields these open ports at the target host:

![](Images/Pasted%20image%2020260728115458.png)

We see several services running on port:
- [ ] 53: DNS
- [x] 80: HTTP web server
- [x] 88: Kerberos (Authentication protocol for AD-environments)
- [x] 135: Microsoft RPC
- [ ] 139: NetBIOS-ssn (used for smb file sharing, technically obsolete)
- [x] 389: LDAP (Lightweight Directory Access Protocol for directory querying.)
- [ ] 443: HTTPS (Encrypted web traffic protocol)
- [x] 445: SMB (Server Message block for file sharing in AD)
- [ ] 464: Kerberos password change/setup service
- [ ] 593: RPC over HTTP
- [ ] 636 and 3268+3269: LDAPS (encrypted LDAP) and global catalog/global catalog secure
- [ ] 3389: RDP (Remote Desktop Protocol) 

We start off by seeing if we can access ldap, rpc or smb with anonymous access.

Anonymous ldap `netexec ldap $target -u '' -p '' --users` yields a userlist. 

![](Images/Pasted%20image%2020260728115510.png)

It also suggests that some of these user has credentials that are set by default:

![](Images/Pasted%20image%2020260728115520.png)

Anonymous smb also yields all users. We save it to a userlist.txt by running: `netexec smb $target -u 'guest' -p '' --rid-brute | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f2 | tee userlist.txt`

## Exploitation

With the userlist, we can attempt asreproasting, which yields 5 asrep hashes:

![](Images/Pasted%20image%2020260728115532.png)

After attempting 3 large wordlists, the asreproasts are unsuccessful. Since we likely have credentials from the ldap query, we can attempt Kerberoasting as well with these credentials.
Kerberoasting also does not yield anything for now:

![](Images/Pasted%20image%2020260728115544.png)

We can attempt password spraying for the open services, one of which yields valid credentials for the RDP service:

![](Images/Pasted%20image%2020260728115554.png)

We successfully connect with RDP and find a user flag on the desktop:

![337](Images/Pasted%20image%2020260728115602.png)

We should check which permissions she has by opening a shell and typing `whoami /all`
And we see that she does not have a lot of permissions:

![](Images/Pasted%20image%2020260728115613.png)

## Post-Exploitation (Privilege Escalation)

We then attempt to run some scripts to check for Privilege escalation. Since we mounted a share with attacking scripts, we must move them to a location that let's us execute it:
`cd C:\Windows\Tasks` and then run
`copy \\TSCLIENT\share\*` 

![](Images/Pasted%20image%2020260728115624.png)

We see that antivirus flagged rubeus.exe and winPEASany.exe, which is correct, although the other scripts did not yield privelege escalation.
  
This means we need to find another route to privilege escalation. We check to see what right our user has pertaining to certificate security settings, using the wiki https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation:

![](Images/Pasted%20image%2020260728115634.png)

Checking the content of the output, we see that the enrollment rights of the certificate are allowed by authenticated users, which our user is a part of:

![539](Images/Pasted%20image%2020260728115643.png)

The output of this also points to possible exploitable vulnerabilities, so we follow ESC1 in the wiki:
![](Images/Pasted%20image%2020260728115656.png)

This vulnerability let's authenticated users request a certificate for anyone in the environment from the CA and the CA supplied me a certificate which authenticates us as the administrator when we show the certificate.

We attempt to obtain the NT hash for the administrator, but are unsuccessful:

![](Images/Pasted%20image%2020260728115705.png)

We can instead attempt to authenticate to a service that grants us admin rights, which we attempt to do with an ldap shell:

![](Images/Pasted%20image%2020260728115713.png)

Since we do not know either our hash or our credentials as the admin user, we add our user to the domain admin group:

![](Images/Pasted%20image%2020260728115722.png)

Verifying that the user got added:

![](Images/Pasted%20image%2020260728115730.png)

We can now hashdump with the user and we attempt to do it against the smb service:

![](Images/Pasted%20image%2020260728115738.png)

Since we now have the administrator hash we should check if we can spawn an admin shell, but it fails for some reason:

![](Images/Pasted%20image%2020260728115748.png)

Since we have a valid path for getting certificates, we can attempt to do that with other domain controller users, in case the administrator user has configurations that does not let us exploit it. We launch bloodhound, fetch the file with our SUSANNA_MCKNIGHT user, ingest it and check for domain controllers in the environment:

![](Images/Pasted%20image%2020260728115756.png)

Let's find the flag, which is likely located on an administrative account. We dump the hashes from all users into a file and also find the hash for Bradley:

![](Images/Pasted%20image%2020260728115805.png)

We then spawn an admin shell:

![](Images/Pasted%20image%2020260728115814.png)


We find the flag in the Administrator desktop:
![404](Images/Pasted%20image%2020260728115825.png)



## Findings and Flags
In this section, finding from various sources gets collected, typically usernames, hashes, credentials or similar.

Domain: thm.local


#### Usernames:Passwords
SUSANNA_MCKNIGHT:CHANGEME2023! (Valid for RDP)
IVY_WILLIS:CHANGEME2023!

#### Usernames:Hashes

**Admin NT hash**
Administrator:500:aad3b435b51404eeaad3b435b51404ee:5565d2f8a32fd35d1c8ee5a8ad198823:::
BRADLEY_ORTIZ:1358:aad3b435b51404eeaad3b435b51404ee:16ec31963c93240962b7e60fd97b495d:::
##### Asrephashes
saved in asrephashes.txt in case of further craching.

## Userflag
User: SUSANNA_MCKNIGHT:CHANGEME2023!
Flag: THM{ENUMERATION_IS_THE_KEY}

# Rootflag
User: BRADLEY_ORTIZ with pass the hash.
Flag: THM{THE_BYPASS_IS_CERTIFIED!}


# Suggested remediation
In this environment, there is several suggested remediations:
- Disable anonymous access (Used to get initial foothold)
- Disable guest accounts
- Remediate specific certificate vulnerability
- Change password distribution methods

It is suggested to review other certificate vulnerabilities as well and put some methodology for creating a validation scheme around it.