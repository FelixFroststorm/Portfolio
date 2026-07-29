#HTB #SMB

# Executive summary
There are many ways to share files between computers and their users. In this case, a service called SMB (Server Message Block) was set up to share files. This is a common way of sharing files in a Windows environment. It was vulnerable due to misconfiguration by allowing a guest user with no credentials to access the files. 




# Enumeration
Nmap was used for initial enumeration, scanning open TCP ports:

![](Images/Pasted%20image%2020260728114425.png)


Notable ports are:
- TCP 139 - Legacy SMB protocol component
- TCP 445 - SMB

### Initial access
We try to check if we are able to list the shares of the network as anonymous (no password):

![](Images/Pasted%20image%2020260728114526.png)

There are 4 shares which is accessible over the network, which we can try to access:

![](Images/Pasted%20image%2020260728114546.png)

The first two fail, and the third one is an administrative shares which we will let be for now, perhaps getting back to it later.

We do get access to the "WorkShares" share:

![](Images/Pasted%20image%2020260728114603.png)

Poking around, we fetch "worknotes.txt" and then we check the other folder James.P:

![](Images/Pasted%20image%2020260728114624.png)

In the upper terminal we get the flag, and in the lower one we write it out, which concludes this machine.


# Remediation / Suggestions
Guest and anonymous access should be disabled. Some suggestions include:
- Set the browsable setting to "no"
- Set the guest ok to "no"

SMB should also bet configured to only allow access to shares after authenticating. 

