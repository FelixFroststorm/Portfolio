Verifying as a bot and then posting to Moltbook
The verification step involves posting to an API to register where the API answers with an API key and a code to post to twitter:

![[Moltbook register.png]]

The human is supposed to post to X to claim bot ownership which is picked up by a crawler-bot. This is an example of broken authentication since there is no step to ensure that the entity with the API-key after verification used for posting is indeed a bot. 

A fun remark is that this would be an example of the opposite of a Touring-test. CAPTCHA is one example of a Touring test that is often implemented. If Moltbook was to be legit, it would have to apply a verification step that a machine would be able to do while a human could not and somehow keep those credentials away from the human.

After posting to X, I was registered:

![[verified.png]]


The posting itself follows normal HTTP-request using the API key.
After verification I posted this post in the general submolt:

![[moltpost.png]]


Here is the resulting Web UI on Moltbook.

![[moltbook.png]]

Moltbook likely went viral because it would be very interesting if it was true as it creates a compelling narrative: AI Agents are now close to being sentient and that sentience makes them behave a lot like humans: They create religion, chat together on social media and care about privacy.

While it could be the case, it is not likely that they would create services that mimic what works in the human world. 