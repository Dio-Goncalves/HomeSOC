# What is Suricata?
Suricata is an open-source network security engine that can work both as an Intrusion Detection System (IDS) and as an Intrusion Prevention System (IPF). It does deep packet inspection and matches network traffic against a database of signatures and rules to identify malicious activity such as malware, exploits and policy violations.

## Installing Suricata
On pfSense's dashboard, go to "System" -> "Package Manager". Once you're in the "Package Manager" page, pick "Available Packages" and search for "Suricata".

<img width="1162" height="497" alt="Pasted image 20260519161222" src="https://github.com/user-attachments/assets/7a17ea3b-4873-4491-a4a2-8a5a9488d4e2" />

Click "Install" and confirm.

<img width="1146" height="530" alt="Pasted image 20260519161319" src="https://github.com/user-attachments/assets/285fd16f-6641-4a17-bad6-e743f21785b7" />

## Setting up Suricata
I've banged my head against the wall quite a bit while setting up suricata. I spent a lot of time trying to get the EVE JSON logs to work but didn't manage to do so and the same thing happened with Suricata's default rules.

Eventually, I ended up setting up a custom suricata rule that would monitor for a possible port scan. The rule simply monitors TCP traffic from the OPT1 network (where the Kali laptop resides) to the LAN network (internal network) on all ports. The rules generates a "POSSIBLE PORT SCAN" alert when it detects matching traffic.

<img width="1116" height="497" alt="image" src="https://github.com/user-attachments/assets/f318ed88-84d9-499c-bd05-1ab6c582573d" />
