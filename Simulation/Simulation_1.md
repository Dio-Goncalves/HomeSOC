# Introduction
The aim of this exercise is to perform a basic attack route while showcasing the capabilities of Splunk to aggregate all the generated telemetry and display the necessary information via search queries or created dashboards in a clean manner.

The attack will consist on your typical network scans, brute-forcing users, lateral movement, privilege escalation, scheduled tasks and kerberoasting. We'll start by going through the attack and then we'll proceed towards the detection side, where we'll look for indicators of each part of the attack and try to reassemble it.

The purpose of this isn't to simulate a realistic black box attack environment, as I'll use known credentials in the form of dictionaries and have intentionally opened ports on the machines to access them. This attack is to merely generate noise for future analysis.

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
   - Simulation 1

# Attack
## Network Reconnaissance
### Local Network Discovery - [T1018](https://attack.mitre.org/techniques/T1018/)
So, from the kali machine, we start by performing an ARP scan. "ARP" means "Address Resolution Protocol" and it is responsible for mapping IP addresses to MAC addresses within a Local Area Network. One major advantage the ARP scan holds over the regular network scans that rely on the ICMP protocol is that the ARP scan operates at layer 2, meaning that it can perform detection even if the target is firewalled or blocking pings.
That said, the command `sudo arp-scan -I eth0 -l` is executed, to enumerate the designated network interface, eth0.

<img width="975" height="268" alt="Pasted image 20260706074943" src="https://github.com/user-attachments/assets/1932e3f8-b7df-4b1c-b03a-298e60206da1" />

### Service Enumeration
Having found the next target, we proceed to enumerate its services with `nmap -sV <ip_address>`. [Nmap](https://nmap.org/book/man.html) is a tool that sends raw IP packets to target hosts and analyzes them to determine what hosts are available on the network, what services the hosts are offering, what operating systems are they running, among other characteristics.

<img width="624" height="176" alt="Pasted image 20260706080010" src="https://github.com/user-attachments/assets/8a92479d-d469-414e-9c6b-02b770b8da29" />

## Initial Access
### SSH Credential Brute Force
Knowing that port 22 is open on this machine, we can start thinking about ideas to gain access to it remotely. In this case, we'll use a known username and a password dictionary to brute-force it with [hydra](https://www.kali.org/tools/hydra/). Hydra is a parallelized network logon cracker used to brute-force or perform dictionary attacks for various network services and protocols. We proceed by executing the command `hydra -l "username" -P dictionary.txt <ip_address> ssh`, finding 1 valid password.

<img width="624" height="69" alt="Pasted image 20260706080351" src="https://github.com/user-attachments/assets/a45bc0d2-3bd1-4847-b38e-ce1c361bdee5" />

Then, we can simply proceed to connect to the target machine via 'ssh username@<ip_address>' and inserting the corrrect password.

## Linux Account Manipulation
Once we're remotely connected to the target, the plan is to generate telemetry by locking an existing user, deleting said user, creating a new user and give that new user admin privileges. Then we'll proceed with creating a cronjob.
To lock an existing user, the command `sudo usermod -L username` can be used. 

To delete a user, we can use the command `sudo userdel -r username`. The `-r` flag deletes said user's home directory and mailbox, if there is one.

To create a new user, we can use the command `sudo adduser username`.

To add a user to sudo grupo, we can use the command `sudo usermod -aG sudo username`. The -a flag serves to append the user to the supplemental groups mentioned by the following '-G' flag, in this case the sudo group.

<img width="549" height="370" alt="Pasted image 20260706081404" src="https://github.com/user-attachments/assets/84db7d3a-c9e9-4c5d-8246-d54261703c67" />

## Linux Persistence
### Scheduled Task Creation
Once we're done with manipulating users, we can proceed with creating a cronjob.  These are scheduled tasks that, like user creation, that can be manipulated for attaining persistence. 

In this case, we'll create a harmless scheduled task that every 5 minutes, will write "Attack Test" to a file in the '/tmp' directory. Executing the command 'crontab -e' to open the cron editor, we then select our preferred choice (nano or vim) and add the line `*/5 * * * * echo "Attack Test" >> /tmp/splunk_cron.log`, referencing the previously mentioned task and then save and exit the file.

To verify if the line was added successfully we can check on our current cronjobs with the command `crontab -l`.

<img width="590" height="547" alt="Pasted image 20260706082238" src="https://github.com/user-attachments/assets/3e823101-6c49-40a3-be5b-8540b41e08e6" />

To clean the cronjobs, the command `crontab -r` can be used.

Once we're done on this host, we'll proceed to scan the next target subnet, the 10.20.20.0/24 internal network.

## Internal Network Reconnaissance
We'll start by targeting the 10.20.20.10 machine, entering now in a Windows environment.

<img width="624" height="426" alt="Pasted image 20260706085108" src="https://github.com/user-attachments/assets/c4888e1a-3baa-4f71-8aad-4fb858cca928" />

The goal here, among other secondary objectives, will be to exploit Windows Active Directory by performing the [Golden Ticket](https://www.crowdstrike.com/en-us/cybersecurity-101/cyberattacks/golden-ticket-attack/) technique. This is an attack that exploits weaknesses in the Kerberos authentication protocol, allowing an adversary to maintain persistent access and move laterally within the network, skipping authentication checks. This allows us to effectively pretend to be any user we want, including domain administrators, by forging TGTs (Ticket Granting Tickets) at will, using the KRBTGT password hash. The KRBTGT user is the KDC's (Key Distribution Center) service user responsible for ticket deployment.

To execute this technique, we'll use the tool [mimikatz](https://github.com/gentilkiwi/mimikatz), a very well-known open-source tool that allows attackers to extract plaintext passwords, hashes, among others and also perform pass-the-hash, pass-the-ticket attacks or build Golden tickets. For this, we'll need to gather some information:  
 - username of the user we want to pretend to be;
 - domain name;
 - domain SID;
 - NTLM hash of the KRBTGT user.

## Credential Dumping
Starting with the NTLM hash of the KRBTGT user, we can perform a credential dumping using [crackmapexec](https://crackmapexec.org/). This is an open-source tool designed to assess Active Directory networks, widely used in post-exploitation and pentesting. We can dump the NTDS database by executing the command `crackmapexec smb 10.20.20.10 -d DOMAIN -u USER -p PASSWORD --ntds`. 

<img width="1060" height="307" alt="Pasted image 20260706092114" src="https://github.com/user-attachments/assets/84ca04e8-cf6a-4fc5-821c-f464f44bd7fa" />

With this, we obtained the NTLM hash of krbtgt. Then, we proceed to connect to the Windows Machine using xfreerdp.

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

### Registry Run Key Modifications
Moving onto the registry key changes, we'll create a key that'll make the system launch notepad.exe when the user logs in. This can be achieved my executing the following command in powershell `reg add HKCU\Software\Microsoft\Windows\CurrentVersion\Run /v Test /t REG_SZ /d "notepad.exe"` where:
- `HKCU`= HKEY_CURRENT_USER, only affects the current user;
- `Software\Microsoft\Windows\CurrentVersion\Run` = Run key, whose values are launched automatically when the user logs in;
- `/v`= value name;
- `/t`= value type;
- `/d` = data stored.
Then following the same principle as the scheduled task, the created key can be deleted with the command `reg delete HKCU\Software\Microsoft\Windows\CurrentVersion\Run /v Test /f`. The `/f` flag can be added to force deletion, not asking for confirmation.

<img width="624" height="45" alt="Pasted image 20260706094228" src="https://github.com/user-attachments/assets/6676fd5e-d004-4675-afa5-e805d16b2168" />

## Powershell Activity
Lastly, we execute the command `Invoke-Expression "Get-Date"` to generate some cmdlet powershell activity.

<img width="337" height="54" alt="Pasted image 20260706094708" src="https://github.com/user-attachments/assets/02305dda-765e-4aeb-aec3-4103907c1ec1" />

## Golden Ticket Attack
Resuming with the Golden Ticket attack, our next step is to pick a user to impersonate and enumerate the domain's SID. By executing the command `net user /domain`, we can see the users that reside in the domain. For this exercise, we'll pick the user Administrator.

Next, we need to enumerate the domain SID. This can be done by executing the command `wmic useraccount where name='%username%' get domain,name,sid`. The domain SID will be the obtained SID, but without the last 4 digits, since these are a reference to the user.

<img width="975" height="336" alt="Pasted image 20260706101924" src="https://github.com/user-attachments/assets/38918b70-2b4a-4478-8709-6307ae548903" />

Having all the information on our side, we can then proceed to use mimikatz to forge a golden ticket in the name of the Administrator user for our current domain.

<img width="975" height="301" alt="Pasted image 20260706102256" src="https://github.com/user-attachments/assets/486278e1-dafe-4179-b3d7-bf3ea0fc0880" />


