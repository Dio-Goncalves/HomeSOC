# Introduction
I've decided to include some Windows machines on this project to be able to work with windows telemetry and active directory. These machines come in the form of a Windows Server 2019, which will act as a Domain Controller, and a Windows 11 endpoint which will simulate a simple user endpoint.

# Setting up the Windows Server VM
Before booting the machine, in virtualbox, this machine will only be working with one network adapter, connected to our internal network.

<img width="388" height="49" alt="image" src="https://github.com/user-attachments/assets/581b4312-3213-4e43-a5c5-b2c2108f9cac" />

After going through the installation, its a good idea to update the OS with the latest updates before proceeding.

To change the name of the machine, the following powershell command can be used:
```
Rename-Computer -NewName "your-name" -Restart
```

## Setting up the network connection
To setup the network connection, we can make use of the "Server Manager" GUI. Once there, head to "Local Server" and choose the network interface we want to setup. Then, once we have the display of the available interfaces, in "Network Connections", we right click the interface we want to setup and click "Properties". Once we are in properties, double click "Internet Protocol Version 4 (TCP/IPv4)".

<img width="1919" height="911" alt="image" src="https://github.com/user-attachments/assets/8210f560-f247-411f-a424-c9d25b7dd1f8" />

Once there, if its not picked yet, pick the option to "Use the following IP address:" as shown in the picture above. This is necessary because no DHCP is being used in this network, so that the machines can have their own static IP, thus having a stable infrastructure.  

Then, pick the desired IP address, just make sure it matches with the network that was previously setup in pfSense for the internal network interface. Same goes with subnet mask, considering you're most likely using a /24 network, the subnet mask will be 255.255.255.0. Since we want to use pfSense as our default gateway, the IP address for the default gateway should be pfSense's IP address in the internal network.

You should end up with something along these lines:  

<img width="372" height="136" alt="image" src="https://github.com/user-attachments/assets/7b9de6e4-df46-4d82-8715-201b36126df1" />

Then, for the DNS server we should set it up with the loopback address, 127.0.0.1. This is important to guarantee stability in case any problem occurs like if the network interface breaks or the server's static IP accidentally changes.  

<img width="369" height="89" alt="image" src="https://github.com/user-attachments/assets/1c56814f-0e2b-43a9-87ad-143f45de395f" />

Once you're done just click "OK".

You can check if the changes were done correctly on powershell using the command:
```
ipconfig /all
```

To change the name of a network adapter in powershell, you can also use the following command:
```
Rename-NetAdapter -Name "original-name" -NewName "new-name"
```

