#  My Engineering Setup & Workflow

##  Overview
I utilize a modular approach to separates my knowledge base from my attack infrastructure, ensuring data persistence, structure and workflow efficiency.

### Host Machine
* **Usage:** Documentation, Knowledge base, Reporting, Git Management, Host system for virtual machine.
Obisidian is mainly used for both documenting, the knowledge base and reporting. 

### Attack Machine
* **OS:** Kali Linux (Virtualized)
* **Role:** Execution of offensive tools and network interaction.

Oracle Virtualbox is used to manage the VM and connection inside it is managed by VPN.


###  Templating and methodology
To ensure consistency and reproducibility across engagements, I utilize a structured logging system. Every engagement begins with a standardized Markdown template. This will likely change over time as my methodology changes and improves.

#### Documentation Structure
1. **Executive summary:** A non-technical description of what happened and why.
2. **Initial setup:** Quick description of setting up the attacker machine to stay organized.
3. **Enumeration:** Enumeration, service discovery, and initial recon.
4.  **Exploitation:** Precise recreation steps of the breach (Proof of Concept).
5.  **Post-Exploitation:** Privilege escalation paths and artifact collection and/or privilege analysis.
6.  **Findings and flags:** Flags, hashes, and sensitive data (stored securely).
7.  **Suggested remediation:** Suggested remediations to harden, secure or reduce the possibility of exploitation. The Mitre attack framework is mainly used for this.

Different engagements may differ slightly depending on what is found and reported. 
I strive to document my findings and thought process so that anyone could follow. I always document as-I-go rather than writing everything at the end. Executive summary and remediation is written after I disconnect from the environment.


##  Version Control & Security
* **Repositories:** I maintain strict separation between active engagements (Private) and public portfolio work (Public).
* **Sanitization:** All public writeups are cleared to be posted. Credentials, hashes and similar sensitive data would not be written in client reports. 
