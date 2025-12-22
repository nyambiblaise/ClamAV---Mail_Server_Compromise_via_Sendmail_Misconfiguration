# Complete Mail Server Compromise via Sendmail Misconfiguration

Target IP: 192.168.218.42

Attacker IP (Kali): 192.168.45.168

### **`Executive Summary`**

**Severity: CRITICAL**

A critical Remote Code Execution vulnerability was identified in the target mail server running Sendmail version 8.13 with ClamAV milter configured in black-hole mode. Through systematic network enumeration and Open-Source Intelligence gathering, we successfully exploited this vulnerability and achieved **root-level access** to the mail server.

This level of compromise represents a complete system takeover, enabling attackers to read all email communications, access sensitive corporate data, pivot to other network systems, launch internal phishing campaigns, install persistent backdoors, and exfiltrate confidential information. Organizations face severe consequences including regulatory compliance violations, financial losses, reputational damage, and potential legal liability.

**Immediate action is required.** The IT Security team must patch Sendmail to the latest version, update ClamAV, reconfigure mail server security controls, and implement the comprehensive remediation steps outlined in the Recommendations section.

### **`Reconnaisance & Enumeration`**

We perform an nmap port scap and identified ports 22, 25, 80, 139, 199 and 445 open. 

![image.png](image.png)

We then performed a service version scan (`nmap -sV 192.168.218.42 -p 22,25,80,139,199,445`) and discovered that the target is running OpenSSH3.8, Sendmail 8.13 and an apache web server.

![image.png](image%201.png)

We did a curl to the server and found a binary code which when decoded yields to `ifyoudontpwnmeuran00b` which we will keep and **may** need in future.

![image.png](image%202.png)

We performed an agressive scan on port 25 (`nmap -sV -A 192.168.218.42 -p 25`) and confirmed that it is running `sendmail v8.13` and the server is propably Linux.

![image.png](image%203.png)

Using netcat in verbose mode(`nc -vv 192.168.218.42 25`), we confirmed that the service running on port 25 is using ESMTP Sendmail

![image.png](image%204.png)

Since we also identified port 199 running a SNMP, we performed an SNMP Check on the port using `snmp-check 192.168.218.42` and discovered that `clamav-milter` is running with **Sendmail** in `black-hole-mode`. 

Our Understanding of what we have discovered so far:

1. Sendmail is a mail transfer agent that is listening on port 25
2. ClamAV is an antivirus engine that also scans email contents for malware
3. Milter stands for **Mail Filter** that inspects SMTP transactions as they happen. It can accept, reject, quarantine, or discard messages.
4. Black-hole-mode means silently discarded without the sender notified.

![image.png](image%205.png)

![image.png](image%206.png)

Alternately, we can continue searching for vulnenrabilities for other discovered services such as SMB, using **Searchsploit and** OSINT, we were able to find an exploit that matches Sendmail and ClamAV with milter running in black-hole-mode.

We found a Remote Code Execution vulnerability ([https://www.exploit-db.com/exploits/4761](https://www.exploit-db.com/exploits/4761)) that exploits Sendmail with clamav-milter vulnerabilities.

![image.png](image%207.png)

Searching on **`Searchsploit` using either commands (`searchsploit sendmail`** or `searchsploit clamav`**),** we found the same vulnerability mapping to a perl file.

![image.png](image%208.png)

![image.png](image%209.png)

At this stage, we have found a vulnerability and an exploit for that vulnerability, we will be moving to the next phase of our attack.

### **`Execution`**

We confirmed that both **wget** and **searchsploit** are mapping to the same file.

Searchsploit: `searchsploit clamav && searchsploit -m multiple/remote/4761.pl`. Here, we mirror the file from the **Exploit-DB** and save it in the current working directory with the name **4761.pl**

![image.png](image%2010.png)

Using **wget (`wget http://exploit-db/download/4761 -O exploit.pl`)**, we download the other exploit 

![image.png](image%2011.png)

Comparing both files using **`diff -y 4761.pl exploit.pl`,** we confirmed that both exploits are the same, so we will proceed with any of them.

![image.png](image%2012.png)

We execute the exploit using **`perl exploit.pl`,** but it requests a host to connect to. Looking at the code, using Static Source Code Analysis, we identified that there is a line saying that **`if ($#ARG[0]!=0)`**, the exploit returns the above-seen message. 

visibly, there are 2 ways we could solve this:

1. Creating a variable $ARGV[0] and assigning it our target IP: **`$ARGV[0]="192.168.218.42";`**the code now becomes:
    
    ![image.png](image%2013.png)
    
2. We pass the target as parameter directly after executing the file: **`perl exploit.pl 192.168.218.42.`**

Both methods will definitely yield the same result. This was tested and confirmed.

### **`Exploitation & Privilege Escalation`**

After executing the payload, we got this:

![image.png](image%2014.png)

From the result, we identified the following:

1. the exploit is trying to attack the target
2. The exploit successfully connected to the target
3. We see a typical **bind shell** and a port 31337. The shell writes **`/bin/sh -i`** which is an interactive root shell to **`/etc/inetd.conf`** using **`root`** and restarted the service. **THIS IS GOLDEN !!!!!**.
4. We checked with nmap again and confirmed that a new service now appears running on port 31337 which was not there on our previous scan. this shows that there is propably a chance that root execution has occured and the best way to check will be to connect to the port using netcat.

![image.png](image%2015.png)

1. Running the bind shell using **`nc 192.168.218.42 31337`**, we were able to have immediate root access confirming the exploit was successful.

![image.png](image%2016.png)

### **`Recommendations`**

1. **Update Antivirus Software** - Ensure ClamAV and all antivirus solutions are updated to the latest versions with current virus definitions and security patches
2. **Block Known Malicious Ports** - Restrict access to high-risk ports like 31337 (commonly used for bind shells) through firewall rules and only allow necessary ports
3. **Patch Sendmail Immediately** - Update Sendmail to the latest version to address known RCE vulnerabilities, especially those affecting versions 8.13 and earlier
4. **Disable Black-Hole Mode** - Configure clamav-milter to notify senders of rejected emails rather than silently discarding them to improve visibility
5. **Implement Network Segmentation** - Isolate mail servers from critical systems to limit potential compromises
6. **Enable Email Authentication** - Deploy SPF, DKIM, and DMARC to prevent email spoofing and reduce attack surface
7. **Monitor SMTP Transactions** - Implement real-time monitoring and alerting for suspicious SMTP activities and unexpected port openings
8. **Harden Service Configurations** - Disable unnecessary services, remove default configurations, and apply the principle of least privilege
9. **Regular Vulnerability Scanning** - Conduct periodic vulnerability assessments and penetration testing to identify exploitable weaknesses
10. **Incident Response Plan** - Develop and maintain a documented plan for responding to mail server compromises and RCE attacks

### **`Why It Matters To Organizations`**

1. **Business Continuity Risk:** Email infrastructure is mission-critical for business operations. A compromised mail server can disrupt communication, halt business processes, and damage productivity across the entire organization.
2. **Data Breach Exposure:** Mail servers contain sensitive corporate communications, customer data, intellectual property, and confidential information. Root-level access (as demonstrated in this exploit) gives attackers unrestricted access to all stored emails and attachments.
3. **Lateral Movement & Escalation:** Once attackers gain root access to a mail server, they can pivot to other systems, install persistent backdoors, and escalate attacks throughout the network infrastructure.
4. **Compliance & Regulatory Impact:** Organizations in regulated industries (healthcare, finance, government) must protect email communications under laws like HIPAA, GDPR, SOX, and PCI-DSS. A breach can result in severe fines, legal liability, and mandatory breach notifications.
5. **Reputation & Trust Damage:** Email compromise can be used to launch phishing campaigns against customers and partners, severely damaging organizational reputation and stakeholder trust.
6. **Financial Consequences:** The average cost of a data breach exceeds millions of dollars when factoring in incident response, forensics, legal fees, regulatory fines, customer compensation, and lost business opportunities.
7. **Silent Attack Vector:** The black-hole mode configuration in this exploit allows attackers to operate undetected, giving them extended time to exfiltrate data, establish persistence, and cause maximum damage before discovery.
