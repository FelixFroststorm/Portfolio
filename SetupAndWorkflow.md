#  My Engineering Setup & Workflow

##  Architectural Overview
I utilize a modular approach to separates my knowledge base from my attack infrastructure, ensuring data persistence, structure and workflow efficiency.

### Host Machine
* **Usage:** Documentation, Knowledge base, Reporting, Git Management, Host system for virtual machine.
Obisidian is mainly used for both documenting, the knowledge base and reporting. 

### Attack Machine
* **OS:** Kali Linux (Virtualized)
* **Role:** Execution of offensive tools and network interaction.
* **Connectivity:** Oracle Virtualbox with mounted drive.


##  The "Golden Template" Methodology
To ensure consistency and reproducibility across engagements, I utilize a structured logging system. Every engagement begins with a standardized Markdown template. This will likely change over time as my methodology changes and improves.

### Documentation Structure
1.  **Enumeration:** Enumeration, service discovery, and initial recon.
2.  **Exploitation:** Precise recreation steps of the breach (Proof of Concept).
3.  **Post-Exploitation:** Privilege escalation paths and artifact collection and/or privilege analysis.
4.  **Exfiltration:** Flags, hashes, and sensitive data (stored securely).
5.  **Suggested remediation:** Suggested remediations to harden, secure or reduce the possibility of exploitation. 

Different engagements may differ slightly depending on what is found and reported. 
I strive to document my findings and thought process so that anyone could follow. I always document as-I-go rather than writing everything at the end.



##  Version Control & Security
* **Repositories:** I maintain strict separation between active engagements (Private) and public portfolio work (Public).
* **Sanitization:** All public writeups are cleared to be posted.
