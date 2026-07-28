#THM #AD 

**OS:** Windows  

## Executive summary

In this box, we exploit vulnerabilities in Active Directory with a technique called Asreproasting and Kerberoasting. We get access to credentials through misconfiguration in the AD environment. After we find usernames, we are able to request crackable password hashes which we can then use to find real credentials, which we in turn use to move inside the environment to progress further. A service user was found that has too elaborate rights which in turn made it possible to become the Administrator of the environment.



## Initial setup
First, we connect over VPN where the target is located. We also create a folder representing the machine we are targeting, go into it and save the ip-address as the variable in our terminal before starting. 

Creating our folder and starting off inside it:
`mkdir THMRazorBlack && cd THMRazorBlack`

## Enumeration

Initial scan: 
![264](Images/Pasted%20image%2020260728115843.png)

We see that this is a Windows computer since it runs Kerberos, ldap and smb. 
We check for anonymous and guest login for the services ldap, rpc and smb. We also check for public shares.

We see that there is a public share called raz0rblack.thm, obtained through anonymous access:
![](Images/Pasted%20image%2020260728115855.png)
  
We are able to establish an anonymous rpc connection, but do not have rights to list all users:
![378](Images/Pasted%20image%2020260728115910.png)

We only have the domain for now, so we ned to see if we can access other services. We attempt to talk to nfs over port 2049 and do a google search to find out how we can do that. We attempt to see if we can mount a the server directory, which we can:
![407](Images/Pasted%20image%2020260728115922.png)

Following the guide, we create a directory and copy its contents into it. After listing the content we get two files, an excel sheet and a flag. sbradley is likely a username.
In the excel file, there is a list of different users and roles, according to the file:
![454](Images/Pasted%20image%2020260728115934.png)

sbradley is likely Steven Bradley, which likely also gives us the username convention. 
We run a script that generates a userlist that includes this naming convention and save it as generated_users.txt
We now have a userlist we can use in the environment for further enumeration.

We attempt Kerbrute and find valid usernames:
![545](Images/Pasted%20image%2020260728115946.png)


## Exploitation

  Now we can attempt asreproasting:
![574](Images/Pasted%20image%2020260728115957.png)
User twilliams is vulnerable to asreproasting, so now we can attempt to crack that hash with johnthe ripper which succeeds, and we have his credentials:
![](Images/Pasted%20image%2020260728120007.png)

We now have two main options:
- Test twilliams credentials
- Attempt  kerberoasting with valid credentials

We attempt kerberoasting first and we are able to get another hash, a kerberos hash for the service account xyan1d3:
![](Images/Pasted%20image%2020260728120017.png)
We can now attempt to crack that hash as well to get credentials for that service account, which we do:
![](Images/Pasted%20image%2020260728120024.png)

We want to test these credentials against open services.

After attempting winrm with the service user xyan1d3, we see that we have shell access:
![](Images/Pasted%20image%2020260728120034.png)

We attempt to spawn a shell win evil-winrm, which succeeds:
![](Images/Pasted%20image%2020260728120044.png)

Since we have a shell, we attempt to go for privilege escalation. When checking which privileges we have, we see that we have SeBackupPrivilege and SeRestorePrivilege enabled. This means that we can copy from the registry to get credentials from the whole system:
**There was an image here, but I made a mistake at some point and lost the image**


## Post-Exploitation (Privilege Escalation)

First, we create a folder on C from the shell on the target:
`mkdir temp` and go into it (C:\\temp)
we then copy from the registry:
`reg save hklm\sam c:\Temp\sam` which results in the sam file and
`reg save hklm\system c:\Temp\system` which results in the system file.

Now we can download it to our attacker by running:
`download sam` and `download system` - It will arrive where the shell was spawned from.

Now, we can run: `impacket-secretsdump -sam sam -system system local` and we get the hash from the administrator.

![](Images/Pasted%20image%2020260728120356.png)
Now we just need to get access to the administrative account. We can attempt a pass the hash attack, which works with impacket-wmiexec (we also attempted psexec):

![](Images/Pasted%20image%2020260728120407.png)


![](Images/Pasted%20image%2020260728120415.png)

We can do anything inside the system now, but for good measure, let's get all the credentials from all of the users:
  

## Findings and Flags
In this section, finding from various sources gets collected, typically usernames, hashes, credentials or similar.
* **User Flag:** THM{ab53e05c9a98def00314a14ccbfa8104}
#### Usernames
lvetrova@raz0rblack.thm (Listed as AD admin)
twilliams@raz0rblack.thm:roastpotatoes
sbradley@raz0rblack.thm
xyan1d3:cyanide9amine5628

#### Hashes
Administrator: :9689931bed40ca5a2ce1218210177f0c