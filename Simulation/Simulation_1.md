# Introduction
The aim of this exercise is to perform a basic attack route while showcasing the capabilities of Splunk to aggregate all the generated telemetry and display the necessary information via search queries or created dashboards in a clean manner.

The attack will consist on your typical network scans, brute-forcing users, lateral movement, privilege escalation, scheduled tasks and kerberoasting. We'll start by going through the attack and then we'll proceed towards the detection side, where we'll look for indicators of each part of the attack and try to reassemble it.

The purpose of this isn't to simulate a realistic black box attack environment, as I'll use known credentials in the form of dictionaries and have intentionally opened ports on the machines to access them. This attack is to merely generate noise for future analysis.

# Contents
1. [Attack](#Attack)
   - [Network Reconnaissance](#Network-Reconnaissance)
   - [Service Enumeration](#Service-Enumeration)
   - [Initial Access](#Initial-Access)
   - [Linux Account Manipulation](#Linux-Account-Manipulation)
   - [Linux Persistence](#Linux-Persistence)
   - [Internal Network Reconnaissance](#Internal-Network-Reconnaissance)
2. [Detection](#Detection)
   - Simulation 1

# Attack
## Network Reconnaissance
So, from the kali machine, we start by performing an ARP scan. "ARP" means "Address Resolution Protocol" and it is responsible for mapping IP addresses to MAC addresses within a Local Area Network. One major advantage the ARP scan holds over the regular network scans that rely on the ICMP protocol is that the ARP scan operates at layer 2, meaning that it can perform detection even if the target is firewalled or blocking pings.
That said, the command `sudo arp-scan -I eth0 -l` is executed, to enumerate the designated network interface, eth0.

<img width="975" height="268" alt="Pasted image 20260706074943" src="https://github.com/user-attachments/assets/1932e3f8-b7df-4b1c-b03a-298e60206da1" />

## Service Enumeration
Having found the next target, we proceed to enumerate its services with `nmap -sV <ip_address>`. [Nmap](https://nmap.org/book/man.html) is a tool that sends raw IP packets to target hosts and analyzes them to determine what hosts are available on the network, what services the hosts are offering, what operating systems are they running, among other characteristics.

<img width="624" height="176" alt="Pasted image 20260706080010" src="https://github.com/user-attachments/assets/8a92479d-d469-414e-9c6b-02b770b8da29" />

## Initial Access
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
Once we're done with manipulating users, we can proceed with creating a cronjob.  These are scheduled tasks that, like user creation, that can be manipulated for attaining persistence. 

In this case, we'll create a harmless scheduled task that every 5 minutes, will write "Attack Test" to a file in the '/tmp' directory. Executing the command 'crontab -e' to open the cron editor, we then select our preferred choice (nano or vim) and add the line `*/5 * * * * echo "Attack Test" >> /tmp/splunk_cron.log`, referencing the previously mentioned task and then save and exit the file.

To verify if the line was added successfully we can check on our current cronjobs with the command `crontab -l`.

<img width="590" height="547" alt="Pasted image 20260706082238" src="https://github.com/user-attachments/assets/3e823101-6c49-40a3-be5b-8540b41e08e6" />

To clean the cronjobs, the command `crontab -r` can be used.

Once we're done on this host, we'll proceed to scan the next target subnet, the 10.20.20.0/24 internal network.

## Internal Network Reconnaissance
