#redis #tcp6379 

**OS:** Linux
**Difficulty:** Easy
       

## Executive summary
Redis (REmote DIctionary Server) is an in-memory database designed for quick retrieval of data, for example prices for a store or a temperature reading off a sensor. It stores data in key-value pairs, for example temperature_london:25. In this case, the server was accessible from the internet without a password, which makes it vulnerable to being taken over.

  

## Enumeration
Scanning yields a single tcp port open, a service named Redis.

* **Open Ports:**

    * TCP 6379 - Redis

* **Notes:**
This service is an in-memory database that is set to 6379 TCP by default. A quick web search show how to interact with the service, and then opening the help directly from the service once we access it.

Bash:
`redis-cli -h $target -p 6379`

![[Pasted image 20260709162740.png]]

  
## Exploitation
**Vulnerability:** Port 6379 is exposed to the internet with the possibility of remotely accessing the service.

There are quite a few commands possible, so we turn to Google to check how we can interact with the service to exfiltrate the data. We find out that listing the keyspace shows us different keys. A key is a label connected to a value. If we run:
`INFO keyspace` 
it lists the indexed databases. 
 ![[Pasted image 20260709164802.png|495]]

Now we want to read those keys, and further googling shows that `SCAN *` lists the different keys:
![[Pasted image 20260709165156.png|301]]
we then go on to read the value with `GET <key>`:
![[Pasted image 20260709165234.png|287]]

The flag is of course our desired goal this time.

Just to push a little further, we should also check if we have write permissions:
![[Pasted image 20260709170320.png|364]]
We see that we do and that we created a key-value pair of temperature:22.

## Loot & Flags

* **Flag:** 03e1d2b376c37ab3f5319922053953eb

## Remediation / Suggestion
A database that is exposed to the internet without a password is a very big risk. Administrator should bind the service exclusively to localhost or a specific internal ip-address so it is not exposed to the internet. By default Redis has no password and as an additional layer the administrator should use the requirepass setting to prevent server hijacking.