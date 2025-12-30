# ClamAV - Mail_Server_Compromise_via_Sendmail_Misconfiguration

Target IP: 192.168.218.42

Attacker IP (Kali): 192.168.45.168

### **`Executive Summary`**

**Severity: CRITICAL**

A critical Remote Code Execution vulnerability was identified in the target mail server running Sendmail version 8.13 with ClamAV milter configured in black-hole mode. Through systematic network enumeration and Open-Source Intelligence gathering, we successfully exploited this vulnerability and achieved **root-level access** to the mail server.

This level of compromise represents a complete system takeover, enabling attackers to read all email communications, access sensitive corporate data, pivot to other network systems, launch internal phishing campaigns, install persistent backdoors, and exfiltrate confidential information. Organizations face severe consequences including regulatory compliance violations, financial losses, reputational damage, and potential legal liability.

**Immediate action is required.** The IT Security team must patch Sendmail to the latest version, update ClamAV, reconfigure mail server security controls, and implement the comprehensive remediation steps outlined in the Recommendations section.

### **`Reconnaisance & Enumeration`**

We perform an nmap port scap and identified ports 22, 25, 80, 139, 199 and 445 open. 
<img width="1165" height="247" alt="image" src="https://github.com/user-attachments/assets/d8ef8f7a-ba96-4c96-af58-78406e414ab5" />


We then performed a service version scan (`nmap -sV 192.168.218.42 -p 22,25,80,139,199,445`) and discovered that the target is running OpenSSH3.8, Sendmail 8.13 and an apache web server.

<img width="1333" height="271" alt="image 1" src="https://github.com/user-attachments/assets/717ffec1-ad8e-4bc3-86f6-2e611ee4b23f" />


We did a curl to the server and found a binary code which when decoded yields to `ifyoudontpwnmeuran00b` which we will keep and **may** need in future.

<img width="1376" height="273" alt="image 2" src="https://github.com/user-attachments/assets/1642af09-2539-41a6-b1ad-042c5be8ad57" />


We performed an agressive scan on port 25 (`nmap -sV -A 192.168.218.42 -p 25`) and confirmed that it is running `sendmail v8.13` and the server is propably Linux.

<img width="1378" height="595" alt="image 3" src="https://github.com/user-attachments/assets/fe5b8f15-6ebe-4754-9e67-1711cde6dde8" />


Using netcat in verbose mode(`nc -vv 192.168.218.42 25`), we confirmed that the service running on port 25 is using ESMTP Sendmail

<img width="1372" height="127" alt="image 4" src="https://github.com/user-attachments/assets/61f4de56-b3aa-463b-9a87-e237ab3cc9af" />


Since we also identified port 199 running a SNMP, we performed an SNMP Check on the port using `snmp-check 192.168.218.42` and discovered that `clamav-milter` is running with **Sendmail** in `black-hole-mode`. 

Our Understanding of what we have discovered so far:

1. Sendmail is a mail transfer agent that is listening on port 25
2. ClamAV is an antivirus engine that also scans email contents for malware
3. Milter stands for **Mail Filter** that inspects SMTP transactions as they happen. It can accept, reject, quarantine, or discard messages.
4. Black-hole-mode means silently discarded without the sender notified.

<img width="1364" height="337" alt="image 5" src="https://github.com/user-attachments/assets/51a9fcee-b535-4869-83fe-14a34278ab9e" />


<img width="1356" height="102" alt="image 6" src="https://github.com/user-attachments/assets/c1f725dd-12ba-4993-85d0-b3a7d9ccf0f9" />


Alternately, we can continue searching for vulnenrabilities for other discovered services such as SMB, using **Searchsploit and** OSINT, we were able to find an exploit that matches Sendmail and ClamAV with milter running in black-hole-mode.

We found a Remote Code Execution vulnerability ([https://www.exploit-db.com/exploits/4761](https://www.exploit-db.com/exploits/4761)) that exploits Sendmail with clamav-milter vulnerabilities.

<img width="1425" height="1065" alt="image 7" src="https://github.com/user-attachments/assets/a9ab35d3-f19e-4591-adcd-eb27e873a926" />


Searching on **`Searchsploit` using either commands (`searchsploit sendmail`** or `searchsploit clamav`**),** we found the same vulnerability mapping to a perl file.

<img width="1338" height="303" alt="image 8" src="https://github.com/user-attachments/assets/db28df2e-8945-4c8a-bcc0-d3e146a9d054" />


<img width="1362" height="853" alt="image 9" src="https://github.com/user-attachments/assets/0de97099-a607-4f4c-ab65-dd212666ee84" />


At this stage, we have found a vulnerability and an exploit for that vulnerability, we will be moving to the next phase of our attack.

### **`Execution`**

We confirmed that both **wget** and **searchsploit** are mapping to the same file.

Searchsploit: `searchsploit clamav && searchsploit -m multiple/remote/4761.pl`. Here, we mirror the file from the **Exploit-DB** and save it in the current working directory with the name **4761.pl**

<img width="1346" height="459" alt="image 10" src="https://github.com/user-attachments/assets/e65df66e-da8f-4039-a6a7-8957aa236037" />


Using **wget (`wget http://exploit-db/download/4761 -O exploit.pl`)**, we download the other exploit 

<img width="1361" height="639" alt="image 11" src="https://github.com/user-attachments/assets/1b6c4881-e7f3-47cf-8843-d3e65ccac8b8" />


Comparing both files using **`diff -y 4761.pl exploit.pl`,** we confirmed that both exploits are the same, so we will proceed with any of them.

<img width="1346" height="609" alt="image 12" src="https://github.com/user-attachments/assets/b8df7756-6d48-4032-9991-5f673293b556" />


We execute the exploit using **`perl exploit.pl`,** but it requests a host to connect to. Looking at the code, using Static Source Code Analysis, we identified that there is a line saying that **`if ($#ARG[0]!=0)`**, the exploit returns the above-seen message. 

visibly, there are 2 ways we could solve this:

1. Creating a variable $ARGV[0] and assigning it our target IP: **`$ARGV[0]="192.168.218.42";`**the code now becomes:
    
  <img width="1368" height="339" alt="image 13" src="https://github.com/user-attachments/assets/5fabdc4f-af25-4d9e-ba9c-b49f0196ccef" />

    
2. We pass the target as parameter directly after executing the file: **`perl exploit.pl 192.168.218.42.`**

Both methods will definitely yield the same result. This was tested and confirmed.

### **`Exploitation & Privilege Escalation`**

After executing the payload, we got this:

<img width="1489" height="488" alt="image 14" src="https://github.com/user-attachments/assets/f0d59248-ccba-4e52-8e7d-168c72a3b7a0" />


From the result, we identified the following:

1. the exploit is trying to attack the target
2. The exploit successfully connected to the target
3. We see a typical **bind shell** and a port 31337. The shell writes **`/bin/sh -i`** which is an interactive root shell to **`/etc/inetd.conf`** using **`root`** and restarted the service. **THIS IS GOLDEN !!!!!**.
4. We checked with nmap again and confirmed that a new service now appears running on port 31337 which was not there on our previous scan. this shows that there is propably a chance that root execution has occured and the best way to check will be to connect to the port using netcat.

<img width="1389" height="277" alt="image 15" src="https://github.com/user-attachments/assets/687a15ad-cbeb-47e9-8a13-f0ab2cbd7b12" />


1. Running the bind shell using **`nc 192.168.218.42 31337`**, we were able to have immediate root access confirming the exploit was successful.

<img width="1347" height="280" alt="image 16" src="https://github.com/user-attachments/assets/f601f481-f43d-464b-b7e0-2e783c23efbf" />


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
