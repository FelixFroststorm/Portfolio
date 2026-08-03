#THM #AD 

**OS:** Windows  
**Date:** 2nd of August 2026
# Attack chain
The attack path in this scenario was:
1. Guest enumeration with SMB
2. Web service that points to Github
3. Leaked credentials on Github
4. Kerberoasting possible with leaked credentials
5. User credentials used to authenticate with RDP
6. Unquoted service path used for privilege escalation
7. Led to system, full compromise

## Initial setup
First, we connect over VPN where the target is located. 

We then create our folder and start it:
```
mkdir THMEnterprise && cd THMEnterprise
```

In each terminal we are using, we also set the environment variable for ip-address. This makes it easier to fetch commands from our knowledge vault and also reduces typos:

```
export target=10.113.175.116
```

We then ping the target to ensure we are able to reach it:

```
ping $target
```


## Enumeration
Our initial scans yields these open ports at the target host:

![203](Images/Pasted%20image%2020260802105805.png)


We start with checking for anonymous and guest access over smb, ldap and rpc. When interacting with smb, we get the domain, which we also save to /etc/hosts:

![457](Images/Pasted%20image%2020260731090907.png)

We are also able to rid brute with the Guest user for SMB, giving us a lot of usernames to try:

![457](Images/Pasted%20image%2020260731112625.png)

ldap and rpc yields nothing, but the guest user has read permission to a share over smb:
 ![](Images/Pasted%20image%2020260731091205.png) 
We see read permissions for both Docs and Users shares. What we would really like is a username, so we check the Users share first. Nothing immediately interesting, so before poking around too much we check Docs and we immediately see interesting files:

![](Images/Pasted%20image%2020260731103245.png)

The files are however password protected. We cat the wordfile and see that there is an encryption hash value that we can attempt to crack. We use office2john to make it into a format that john can crack:

![](Images/Pasted%20image%2020260731103755.png)

This process could use a little help from additional compute power that the attacker machine does not have access to, so we offload this task to our host system with hashcat by putting the office2013 hash into a .txt file and feeding it to hashcat for Windows:

![457](Images/Pasted%20image%2020260731110838.png)

We found match in the rockyou wordlist, so we turn our attention towards the found usernames which we earlier saved to users.txt.

We check with Kerbrute if the usernames we found are currenty valid, which they are:

![404](Images/Pasted%20image%2020260731140712.png)

We now attempt asreproasting. which fails:

![615](Images/Pasted%20image%2020260801093747.png)

We still do not have any valid credentials and after rabbit holing on the password protected files for awhile, I decide to look up if these files are the way in. It turns out, they are not. The correct solution would be to interact with other ports, find a web server that points to an atlassian page that has info that they have moved from Atlassian to Github. A Google search towards this github and digging through commits, reveals that one of the users in our userlist has commited valid credentials at some point:

![422](Images/Pasted%20image%2020260801103519.png)

# Exploitation

  We should of course test these credentials right away, starting with kerberoasting, which works:

![482](Images/Pasted%20image%2020260801104931.png)

So now we attempt to crack it with john, which yields a password:

![](Images/Pasted%20image%2020260801105021.png)

And then we attempt those right away. We go through winrm, smb, ldap and finally rdp that shows that we have permission to spawn a shell:

![](Images/Pasted%20image%2020260801105233.png)

We open an rdp session and find the user flag on the desktop as we log on:

![](Images/Pasted%20image%2020260801105714.png)

  
  

# Post-Exploitation (Privilege Escalation)

Now our goal is to escalate privileges. We start by running the PowerUp. To do that, we need to transfer it to our target machine. We start up a http service on our attacker machine and on the victim we attempt to run the script from memory and pipe the results into power.log:

![](Images/Pasted%20image%2020260802092540.png)

As we see, there is a possible way for us to run an executable due to unquoted service path. This is a vulnerability that works because Windows looks recursively in each directory (because it is unquoted) for an executable. 
First  C:\  
then C:\Program Files (x86)
then C:\Program Files (x86)\Zero Tier 
and so on. 

There is also a dll abuse, but we attempt this first. For this to work, we need a writable directory somewhere in that path. We attempt directory by directory until it works:

![587](Images/Pasted%20image%2020260802093712.png)

That means that we are able to write to the Zero Tier folder within program files. On to checking the service, which is currently stopped:

![281](Images/Pasted%20image%2020260802094140.png)

If we are able to start the service again, Windows will start our injected executable in a folder above it to give us a shell with possibly elevated privileges. First we create the payload and put it into our serving folder:

![](Images/Pasted%20image%2020260802100543.png)

Then we upload it to our victim:

![](Images/Pasted%20image%2020260802102431.png)

We then set up a listener on our attacker machine:

![](Images/Pasted%20image%2020260802102704.png)

And then we start the service, which yields an administrative shell:

![](Images/Pasted%20image%2020260802102835.png)

We then go to the desktop and find our final flag:

![592](Images/Pasted%20image%2020260802102946.png)





## Findings and Flags
In this section, finding from various sources gets collected, typically usernames, hashes, credentials or similar.

Userflag: THM{ed882d02b34246536ef7da79062bef36}

Administrator flag: THM{1a1fa94875421296331f145971ca4881}

#### Usernames:Passwords
bitbucket:littleredbucket
nik:ToastyBoi!

