# Introduction
The aim of this exercise is to perform a basic attack route while showcasing the capabilities of Splunk to aggregate all the generated telemetry and display the necessary information via search queries or created dashboards in a clean manner.

The attack will consist on your typical network scans, brute-forcing users, lateral movement, privilege escalation, scheduled tasks and kerberoasting. We'll start by going through the attack and then we'll proceed towards the detection side, where we'll look for indicators of each part of the attack and try to reassemble it.

The purpose of this isn't to simulate a realistic black box attack environment, as I'll use known credentials in the form of dictionaries and have intentionally opened ports on the machines to access them. This attack is to merely generate noise for future analysis on Splunk.

Two external laptops will be added for this attack. One running native Debian OS, which will be the initial attack target and the second one will be running a Kali VM which will be used to attack. These two machines will be operating on a different network together. Check the Lab Diagram in [architecture](architecture) for reference.

# Contents
1. [Attack](#Attack)
   - [Network Reconnaissance](#Network-Reconnaissance)
     - [Local Network Discovery](#Local-Network-Discovery)
     - [Service Enumeration](#Service-Enumeration)
   - [Initial Access](#Initial-Access)
     - [SSH Credential Brute Force](#SSH-Credential-Brute-Force)
   - [Linux Account Manipulation](#Linux-Account-Manipulation)
   - [Linux Persistence](#Linux-Persistence)
     - [Scheduled Task Creation](#Scheduled-Task-Creation)
   - [Internal Network Reconnaissance](#Internal-Network-Reconnaissance)
   - [Credential Dumping](#Credential-Dumping)
   - [Windows Persistence](#Windows-Persistence)
     - [Scheduled Task Creation](#Scheduled-Task-Creation)
     - [Registry Run Key Modifications](#Registry-Run-Key-Modifications)
   - [Powershell Activity](#Powershell-Activity)
   - [Golden Ticket Attack](#Golden-Ticket-Attack)
2. [Detection](#Detection)
   - [IDS Dashboard](#IDS-Dashboard)
   - [Authentication Dashboard](#Authentication-Dashboard)
   - [Endpoint Activity Dashboard](#Endpoint-Activity-Dashboard)

# Attack
## Network Reconnaissance
### Local Network Discovery
So, from the kali machine, we start by performing an ARP scan. "ARP" means "Address Resolution Protocol" and it is responsible for mapping IP addresses to MAC addresses within a Local Area Network. One major advantage the ARP scan holds over the regular network scans that rely on the ICMP protocol is that the ARP scan operates at layer 2, meaning that it can perform detection even if the target is firewalled or blocking pings.
That said, the command `sudo arp-scan -I eth0 -l` is executed, to enumerate the designated network interface, eth0.

<img width="975" height="268" alt="Pasted image 20260706074943" src="https://github.com/user-attachments/assets/1932e3f8-b7df-4b1c-b03a-298e60206da1" />

To reference this technique, we can use [MITRE ATT&CK T1018 - Remote System Discovery](https://attack.mitre.org/techniques/T1018/).

### Service Enumeration
Having found the next target, we proceed to enumerate its services with `nmap -sV <ip_address>`. [Nmap](https://nmap.org/book/man.html) is a tool that sends raw IP packets to target hosts and analyzes them to determine what hosts are available on the network, what services the hosts are offering, what operating systems are they running, among other characteristics.

<img width="624" height="176" alt="Pasted image 20260706080010" src="https://github.com/user-attachments/assets/8a92479d-d469-414e-9c6b-02b770b8da29" />

To reference this technique, we can use [MITRE ATT&CK T1046 - Network Service Discovery](https://attack.mitre.org/techniques/T1046/).

## Initial Access
### SSH Credential Brute Force
Knowing that port 22 is open on this machine, we can start thinking about ideas to gain access to it remotely. In this case, we'll use a known username and a password dictionary to brute-force it with [hydra](https://www.kali.org/tools/hydra/). Hydra is a parallelized network logon cracker used to brute-force or perform dictionary attacks for various network services and protocols. We proceed by executing the command `hydra -l "username" -P dictionary.txt <ip_address> ssh`, finding 1 valid password.

<img width="624" height="69" alt="Pasted image 20260706080351" src="https://github.com/user-attachments/assets/a45bc0d2-3bd1-4847-b38e-ce1c361bdee5" />

To reference this technique, we can use [MITRE ATT&CK T1110.001 - Brute Force: Password Guessing](https://attack.mitre.org/techniques/T1110/001/).

Then, we can simply proceed to connect to the target machine via 'ssh username@<ip_address>' and inserting the corrrect password.

To reference this technique, we can use [MITRE ATT&CK T1078.003 - Valid Accounts: Local Accounts](https://attack.mitre.org/techniques/T1078/003/) and [MITRE ATT&CK T1021.004 - Remote Services: SSH](https://attack.mitre.org/techniques/T1021/004/).

## Linux Account Manipulation
Once we're remotely connected to the target, the plan is to generate telemetry by locking an existing user, deleting said user, creating a new user and give that new user admin privileges. Then we'll proceed with creating a cronjob.
To lock an existing user, the command `sudo usermod -L username` can be used. 

To delete a user, we can use the command `sudo userdel -r username`. The `-r` flag deletes said user's home directory and mailbox, if there is one.

To reference these two technique, we can use [MITRE ATT&CK T1531 - Account Access Removal](https://attack.mitre.org/techniques/T1531/).

To create a new user, we can use the command `sudo adduser username`.

To reference this technique, we can use [MITRE ATT&CK T1136.001 - Create Account: Local Account](https://attack.mitre.org/techniques/T1136/001/).

To add a user to sudo grupo, we can use the command `sudo usermod -aG sudo username`. The -a flag serves to append the user to the supplemental groups mentioned by the following '-G' flag, in this case the sudo group.

<img width="549" height="370" alt="Pasted image 20260706081404" src="https://github.com/user-attachments/assets/84db7d3a-c9e9-4c5d-8246-d54261703c67" />

To reference this technique, we can use [MITRE ATT&CK T1097.007 - Account Manipulation: Additional Local or Domain Groups](https://attack.mitre.org/techniques/T1098/007/).

## Linux Persistence
### Scheduled Task Creation
Once we're done with manipulating users, we can proceed with creating a cronjob.  These are scheduled tasks that, like user creation, that can be manipulated for attaining persistence. 

In this case, we'll create a harmless scheduled task that every 5 minutes, will write "Attack Test" to a file in the '/tmp' directory. Executing the command 'crontab -e' to open the cron editor, we then select our preferred choice (nano or vim) and add the line `*/5 * * * * echo "Attack Test" >> /tmp/splunk_cron.log`, referencing the previously mentioned task and then save and exit the file.

To verify if the line was added successfully we can check on our current cronjobs with the command `crontab -l`.

<img width="590" height="547" alt="Pasted image 20260706082238" src="https://github.com/user-attachments/assets/3e823101-6c49-40a3-be5b-8540b41e08e6" />

To clean the cronjobs, the command `crontab -r` can be used.

To reference this technique, we can use [MITRE ATT&CK T1053.003 - Scheduled Task/Job: Cron](https://attack.mitre.org/techniques/T1053/003/).

Once we're done on this host, we'll proceed to scan the next target subnet, the 10.20.20.0/24 internal network.

## Internal Network Reconnaissance
We'll start by targeting the 10.20.20.10 machine, entering now in a Windows environment.

<img width="624" height="426" alt="Pasted image 20260706085108" src="https://github.com/user-attachments/assets/c4888e1a-3baa-4f71-8aad-4fb858cca928" />

To reference this technique, we can use [MITRE ATT&CK T1018 - Remote System Discovery](https://attack.mitre.org/techniques/T1018/).

The goal here, among other secondary objectives, will be to exploit Windows Active Directory by performing the [Golden Ticket](https://www.crowdstrike.com/en-us/cybersecurity-101/cyberattacks/golden-ticket-attack/) technique. This is an attack that exploits weaknesses in the Kerberos authentication protocol, allowing an adversary to maintain persistent access and move laterally within the network, skipping authentication checks. This allows us to effectively pretend to be any user we want, including domain administrators, by forging TGTs (Ticket Granting Tickets) at will, using the KRBTGT password hash. The KRBTGT user is the KDC's (Key Distribution Center) service user responsible for ticket deployment.

To execute this technique, we'll use the tool [mimikatz](https://github.com/gentilkiwi/mimikatz), a very well-known open-source tool that allows attackers to extract plaintext passwords, hashes, among others and also perform pass-the-hash, pass-the-ticket attacks or build Golden tickets. For this, we'll need to gather some information:  
 - username of the user we want to pretend to be;
 - domain name;
 - domain SID;
 - NTLM hash of the KRBTGT user.

## Credential Dumping
Starting with the NTLM hash of the KRBTGT user, we can perform a credential dumping using [crackmapexec](https://crackmapexec.org/). This is an open-source tool designed to assess Active Directory networks, widely used in post-exploitation and pentesting. We can dump the NTDS database by executing the command `crackmapexec smb 10.20.20.10 -d DOMAIN -u USER -p PASSWORD --ntds`. 

<img width="1060" height="307" alt="Pasted image 20260706092114" src="https://github.com/user-attachments/assets/84ca04e8-cf6a-4fc5-821c-f464f44bd7fa" />

To reference this technique, we can use [MITRE ATT&CK T1003.003 - OS Credential Dumping:NTDS](https://attack.mitre.org/techniques/T1003/003/).

With this, we obtained the NTLM hash of krbtgt. Then, we proceed to connect to the Windows Machine using xfreerdp.

To reference this technique, we can use [MITRE ATT&CK T1021.001 - Remote Services: Remote Desktop Protocol](https://attack.mitre.org/techniques/T1021/001/).

## Windows Persistence
Before proceeding with the Golden Ticket Technique, we'll sidetrack a bit to generate some additional telemetry. We'll quickly create a scheduled task and do some registry key changes.

### Scheduled Task Creation
Starting with the scheduled task, this will be a harmless demonstration of a task, named "TestTask", that will run notpad.exe program once, at a given time, in this case at 23:59. In a powershell terminal, we execute the command `schtasks /create /tn "TestTask" /tr "notepad.exe" /sc once /st 23:59` to create the new task. To easily memorize the used flags, we can think of them as:
- `/tn`= task name;
- `/tr`= to run;
- `/sc`= schedule or schedule type;
- `/st`= start time.
Once the task is created, it can be deleted with the command `schtasks /delete /tn "TestTask"`.

<img width="624" height="81" alt="Pasted image 20260706093449" src="https://github.com/user-attachments/assets/bb471607-089c-4e3f-bbf3-8d7e2f1857be" />

To reference this technique, we can use [MITRE ATT&CK T1053.005 - Scheduled Task/Job: Scheduled Task](https://attack.mitre.org/techniques/T1053/005/).

### Registry Run Key Modifications
Moving onto the registry key changes, we'll create a key that'll make the system launch notepad.exe when the user logs in. This can be achieved my executing the following command in powershell `reg add HKCU\Software\Microsoft\Windows\CurrentVersion\Run /v Test /t REG_SZ /d "notepad.exe"` where:
- `HKCU`= HKEY_CURRENT_USER, only affects the current user;
- `Software\Microsoft\Windows\CurrentVersion\Run` = Run key, whose values are launched automatically when the user logs in;
- `/v`= value name;
- `/t`= value type;
- `/d` = data stored.
Then following the same principle as the scheduled task, the created key can be deleted with the command `reg delete HKCU\Software\Microsoft\Windows\CurrentVersion\Run /v Test /f`. The `/f` flag can be added to force deletion, not asking for confirmation.

<img width="624" height="45" alt="Pasted image 20260706094228" src="https://github.com/user-attachments/assets/6676fd5e-d004-4675-afa5-e805d16b2168" />

To reference this technique, we can use [MITRE ATT&CK T1547.001 - Boot or Logon Autostart Execution: Registry Run Keys / Startup Folder](https://attack.mitre.org/techniques/T1547/001/).

## Powershell Activity
Lastly, we execute the command `Invoke-Expression "Get-Date"` to generate some cmdlet powershell activity.

<img width="337" height="54" alt="Pasted image 20260706094708" src="https://github.com/user-attachments/assets/02305dda-765e-4aeb-aec3-4103907c1ec1" />

## Golden Ticket Attack
Resuming with the Golden Ticket attack, our next step is to pick a user to impersonate and enumerate the domain's SID. By executing the command `net user /domain`, we can see the users that reside in the domain. For this exercise, we'll pick the user Administrator.

To reference this technique, we can use [MITRE ATT&CK T1087.002 - Account Discovery: Domain Account](https://attack.mitre.org/techniques/T1087/002/).

Next, we need to enumerate the domain SID. This can be done by executing the command `wmic useraccount where name='%username%' get domain,name,sid`. The domain SID will be the obtained SID, but without the last 4 digits, since these are a reference to the user.

<img width="975" height="336" alt="Pasted image 20260706101924" src="https://github.com/user-attachments/assets/38918b70-2b4a-4478-8709-6307ae548903" />

Having all the information on our side, we can then proceed to use mimikatz to forge a golden ticket in the name of the Administrator user for our current domain.

<img width="975" height="301" alt="Pasted image 20260706102256" src="https://github.com/user-attachments/assets/486278e1-dafe-4179-b3d7-bf3ea0fc0880" />

To reference this technique, we can use [MITRE ATT&CK T1558.001 - Steal or Forge Kerberos Tickets: Golden Ticket](https://attack.mitre.org/techniques/T1558/001/).

# Detection
## IDS Dashboard
Starting with our Dashboards, we can head to the Suricata dashboard where we can see the alerts generated by the port scan done with Nmap. 

<img width="1874" height="402" alt="Pasted image 20260706110953" src="https://github.com/user-attachments/assets/3c0ea2d5-1e83-4cd9-9cce-6a3baa4dd9c8" />
<img width="1853" height="377" alt="Pasted image 20260706111007" src="https://github.com/user-attachments/assets/556b032b-b2e3-468d-8769-c0db9bdd6419" />
<img width="1849" height="571" alt="Pasted image 20260706111051" src="https://github.com/user-attachments/assets/a2a66cce-dcd3-4781-9f98-92588d9278c8" />
<img width="1850" height="174" alt="Pasted image 20260706111118" src="https://github.com/user-attachments/assets/68906e59-c821-4352-af97-e28203691df8" />

It is possible to see a very large amount of alerts generated by the Nmap scan via the Suricata rule that was created ( POSSIBLE PORT SCAN ) to detect port scans. We can see the IP of the culprit, 10.20.40.103, the kali linux machine, on the top scanning hosts table, aswell as the IPs of all the targeted hosts on the table next to it.

## Authentication Dashboard
Moving onto our Authentication Dashboards, we can very easily see the authentication attempts done in the attack, aswell as changes done to users or their credentials.

<img width="1872" height="403" alt="Pasted image 20260706112036" src="https://github.com/user-attachments/assets/49ced4e8-5aaf-4312-b68d-6ffa244be119" />
<img width="1863" height="552" alt="Pasted image 20260706112118" src="https://github.com/user-attachments/assets/9e6eb6d4-266e-47af-8b69-9efa7b1f9415" />
<img width="1856" height="254" alt="Pasted image 20260706112212" src="https://github.com/user-attachments/assets/d0694ab6-59fd-4c01-8e76-f910eae30892" />

Another good thing to track is mimikatz execution.

<img width="916" height="349" alt="Pasted image 20260706113409" src="https://github.com/user-attachments/assets/258a28c3-c9ca-4362-8671-9a4a6df444f3" />

Changing our focus to Linux, we can see user management activity.

<img width="1850" height="374" alt="Pasted image 20260706114029" src="https://github.com/user-attachments/assets/4f69a860-e1c1-4d5d-931e-0fd914c086e5" />

## Endpoint Activity Dashboard
Moving onto endpoint activity dashboard, for Windows, we have the Scheduled Task Creation table.

<img width="1498" height="229" alt="Pasted image 20260706114635" src="https://github.com/user-attachments/assets/68cdcb95-6be2-4524-bd28-b2226d527199" />

The Suspicious Powershell Commands, showing the user creation, deletion and the Invoke-Expression command.

<img width="1492" height="442" alt="Pasted image 20260706114739" src="https://github.com/user-attachments/assets/4feb9fc1-1123-41a9-a195-50a74d048ede" />
<img width="1824" height="396" alt="Pasted image 20260706114921" src="https://github.com/user-attachments/assets/959536f9-a509-4ba3-b446-ac11c32200ce" />

The Parent-Child Process Relationship table that shows the commands executed for scheduled task creation and deletion, user manipulation and AD discovery, together with the respective User and Parent Image for full traceability.

<img width="1858" height="477" alt="Pasted image 20260706114839" src="https://github.com/user-attachments/assets/c579022f-a717-4673-a4e7-3e6c18fddf8d" />
<img width="1843" height="176" alt="Pasted image 20260706114851" src="https://github.com/user-attachments/assets/166e4508-8d3e-4e8a-9333-aec4e5f518d4" />

The Registry Key table that tracked the changes made on the registry.

<img width="1847" height="220" alt="Pasted image 20260706114944" src="https://github.com/user-attachments/assets/44e91bff-3def-404b-bc09-2a0097f66358" />

Changing the focus to Linux, we have a Sensitive Command Execution table and Sudo Activity table, that both tracked the changes to users, since `sudo` was used on the commands and also, sensitive commands were executed at the same time.

<img width="1850" height="375" alt="Pasted image 20260706115123" src="https://github.com/user-attachments/assets/ea97d857-1a5d-438e-8c28-acd2f94b405a" />

Finally, the Cron Changes table that tracked the changes made initially on the Debian machine.

<img width="1851" height="324" alt="Pasted image 20260706115211" src="https://github.com/user-attachments/assets/c4258225-d519-4c85-8b94-ff980b18d45a" />






