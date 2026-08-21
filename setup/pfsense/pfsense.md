# Introduction
The chosen firewall for this project was pfSense. I've chosen it because I used it in previous occasions but there are alternatives, like the fully open-source OPNsense. For the next version of this project, I'll actually want to try out OPNsense to experiment with its automation capabilities via REST API.   

Due to time restraints, I've simply experimented with Suricata's rules, since I was looking for something to that could detect port scans, which would be the base for my first attack simulation. In this project I didn't really get to experiment with proper Firewall rules and things like that. I have that pending for the second version of this project, where I'll have no time restraints. 

Also, I'm not an expert in network, what I'm about to show here was what worked for me. I'm aware that there are areas that can be improved and I have pending improvements for the second version of this project, where I plan to dive deeper into each topic and really take my time with it.

# Setting up the pfSense VM
In VirtualBox, before booting the VM, the networks had to be setup.

<img width="518" height="82" alt="image" src="https://github.com/user-attachments/assets/1bcec9ee-aff1-440d-87a4-29fc3c50eb5c" />

Starting with the first bridged network adapter, this will be the one responsible for giving an internet connection to the environment, effectively connecting pfSense to the physical network. **This will be pfSense's WAN interface.**  

The second interface is the internal network, this is where the windows machines and Linux server will reside. These machines can see and communicate with each other but they can not reach the exterior physical network. **This will be pfSense's LAN interface.**  

The third interface is another bridged network adapter. This is because I had two physical devices that I wanted to connect to the whole environment, a laptop that would be a Debian endpoint and a laptop running Kali Linux on a VM, so another bridged interface seemed logical.**This will be pfSense's OPT1 interface.** 

*Observation*: In order to run two bridged adapters on your host computer, you need to have 2 NICs, my solution was getting a USB Ethernet Adapter like [this](https://www.tp-link.com/en/home-networking/usb-ethernet-adapter/ue300c/) in order to have a second NIC to assign to the second bridged interface.

## Why mixing bridged networks and internal networks?
This allows pfSense to sit between the real network and the isolated lab internal network. This is exactly the kind of behavior that we want from it, allowing our VMs to access the internet through pfSense, while keeping the lab network separated from the physical network.

## Setting up pfSense
Once pfSense VM boots, you'll be prompted through the installation. Afterwards you'll be prompted to setup your first network interface. Here you can first setup the WAN interface and then access pfSense's GUI from your host pc's browser, using pfSense's assigned IP. From the GUI its much easier to setup the remaining two interfaces (LAN and OPT1).

To keep a stable lab structure I only have the DHCP on for the first interface. For the other two interfaces, I want to have a static IP since I'll refer to these IPs while setting up other machines and also so I can use any of these IPs to access pfSense's GUI via browser. 

Also, since there'll be prompts for it on the setup, there is no use in setting up IPv6.

Once you're done you should have something like this:

<img width="455" height="62" alt="image" src="https://github.com/user-attachments/assets/e6d889da-9fcd-49d6-8d13-3f082bc79e16" />


# What is Suricata?
Suricata is an open-source network security engine that can work both as an Intrusion Detection System (IDS) and as an Intrusion Prevention System (IPF). It does deep packet inspection and matches network traffic against a database of signatures and rules to identify malicious activity such as malware, exploits and policy violations.

## Installing Suricata
On pfSense's dashboard, go to "System" -> "Package Manager". Once you're in the "Package Manager" page, pick "Available Packages" and search for "Suricata".

<img width="1162" height="497" alt="Pasted image 20260519161222" src="https://github.com/user-attachments/assets/7a17ea3b-4873-4491-a4a2-8a5a9488d4e2" />

Click "Install" and confirm.

<img width="1146" height="530" alt="Pasted image 20260519161319" src="https://github.com/user-attachments/assets/285fd16f-6641-4a17-bad6-e743f21785b7" />

## Setting up Suricata
