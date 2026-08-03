#HTB #AD 

**OS:** Linux/Windows  
**Date:** 3rd of August 2026

## Initial setup
First, we connect over VPN where the target is located. 

We then create our folder and start it:
```
mkdir HTBForest && cd HTBForest
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

![](Images/Pasted%20image%2020260803084406.png)

We see that we have at least two web servers at port 5985 and 47001 which we note.

We start by checking with guest account and anonymous access over smb, ldap and rpc. We find nothing with SMB, but both ldap and rpc lists users, here from rpc, which we save into users.txt:

![311](Images/Pasted%20image%2020260803085237.png)

We also note that the domain is htb.local (both ldap and smb lists it). We put it in /etc/hosts together with the ip-address.
We remove leading and trailing characters from the file so that once usernames remain. We now want to check validity of usernames and attempt asreproasting, which succeeds for user svc-alfresco:

![432](Images/Pasted%20image%2020260803085816.png)

We attempt to crack it with john, which we do:

![](Images/Pasted%20image%2020260803085926.png)

We attempt to authenticate to running services, and find that we are able to spawn a shell with winrm:

![](Images/Pasted%20image%2020260803090220.png)

We attempt to spawn a shell, which we do:

![526](Images/Pasted%20image%2020260803090432.png)

We fetch the flag:

![](Images/Pasted%20image%2020260803090607.png)

and then we want to look to escalate privileges. We check which privileges this user has:

![](Images/Pasted%20image%2020260803092644.png)


We can also start Bloodhound to see if there are paths for us to take. We start it, download the file with ldap and ingest it into bloodhound:

![](Images/Pasted%20image%2020260803102158.png)

We open a query, add svc-alfresco as an owned used and check for shortest path to domain admin:

![556](Images/Pasted%20image%2020260803103140.png)

We attempt to do the chain, adding the user to Exchange Windows permissions group. 

## Exploitation

So now we create a user which we in turn give the Exchange Windows Permissions group membership to - And also to svc-alfresco, we also verify it afterwards:

![](Images/Pasted%20image%2020260803151417.png)

We see in Bloodhound that dacledit.py can be used, which is a part of the impacket suite. We utilize that and give DCSync rights to the user svc-alfresco:

![](Images/Pasted%20image%2020260803153323.png)

  
## Post-Exploitation (Privilege Escalation)
After these rights are given, we attempt to dump all hashes, which we succeed at:

  ![572](Images/Pasted%20image%2020260803161549.png)

Now we can pass the hash to spawn a shell, which we do:

![463](Images/Pasted%20image%2020260803161810.png)

That concludes this box.


