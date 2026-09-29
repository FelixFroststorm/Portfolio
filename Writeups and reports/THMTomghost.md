**OS:** Linux  
**Date:** 25th of September 2026

## Enumeration
Our initial scans yields these open ports at the target host:

![532](Images/Pasted%20image%2020260925073122.png)

Let's visit the http-server at 8080.
It is an instance of Tomcat version 9.0.30:

![](Images/Pasted%20image%2020260925073235.png)

A quick search on Searchsploit shows a lot of vulnerabilities for Tomcat, but not for this exact version. We also search for ajp13, but searchsploit does not find anything.

![411](Images/Pasted%20image%2020260925073659.png)

We do a quick google search and find CVE-2020-1938, which is a critical file read and inclusion vulnerability known as Ghostcat. Looks promising.

![](Images/Pasted%20image%2020260925073751.png)

Let's see if Metasploit has an included script to help us:

![](Images/Pasted%20image%2020260925073907.png)

It does. 
## Exploitation
All it needs is to set the ip-options and we can then run it:

![](Images/Pasted%20image%2020260925080031.png)

![](Images/Pasted%20image%2020260925080118.png)

It reveals manager username and password.

Let's attempt to log onto the console. 

![](Images/Pasted%20image%2020260925080403.png)

When attempting to get to the login page of Tomcat, it won't let us get into the login screen. There are likely settings that whitelist hosts that can do this. Since we have credentials, it is worth spraying them across other services as well and on this service, there is SSH running.

![465](Images/Pasted%20image%2020260925080631.png)

And we have initial access to this host.

We poke around and find the userflag:

![](Images/Pasted%20image%2020260925080938.png)

We check if we have sudo right on skyfuck:

![](Images/Pasted%20image%2020260925082527.png)

And we do not.
Now we want to escalate privileges.
  
## Post-Exploitation (Privilege Escalation)

When we landed on the server as skyfuck, there were two interesting files in that folder.

![](Images/Pasted%20image%2020260925084108.png)

.pgp is a file that says "pretty good privacy" that has a private key. We offload the file to get to offline cracking:

![](Images/Pasted%20image%2020260925084225.png)

We do the same thing for the credential.gpg.

First we need to get the private key into a format that john will understand, so we run:
```
gpg2john tryhackme.asc > hash
```

And then:
```
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

![](Images/Pasted%20image%2020260925084441.png)

Now we have the passphrase for the php file, which we can then decrypt:

![](Images/Pasted%20image%2020260925084637.png)

We can now attempt swapping to merlin to see if he has more credentials.

We see that merlin is not root, but he is able to run as root with no password in /usr/bin/zip:

![](Images/Pasted%20image%2020260925084902.png)

We use GTFOBins to exploit this:
```
sudo zip exploit.zip /tmp/ -T --unzip-command="sh -c /bin/sh"
```

We then get a root shell:

![](Images/Pasted%20image%2020260925090300.png)
