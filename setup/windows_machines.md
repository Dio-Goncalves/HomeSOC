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

## Adding a new user to the Domain Admin group
I think that its generally a good idea to avoid using the Administrator user for general use and only keep it for what its strictly necessary, much like the root user for Linux systems.

Considering that we'll be covering Group Policy Management, in this segment a quick rundown will be done on how to add a new user to the Domain Admins Active Directory Group. Then, the Policy Management part, and other Active Directory management tasks, can be done with the newly added user.

Start by searching "Active Directory Users and Computers" and access it.

<img width="843" height="648" alt="image" src="https://github.com/user-attachments/assets/f4d62343-9e11-42e3-8fa1-b3eb6a4f627e" />

Then, access the "Users" folder.

<img width="753" height="528" alt="Pasted image 20260514004658" src="https://github.com/user-attachments/assets/876fc7f6-ec42-4396-a2af-e612a39a3663" />

Next, on the right column, double click on "Domain Admins", go to "Members" and click "Add...". Write the username that you wish to add and click on "Check Name. Select the username and click "OK". In the example below, I added the user "Admin".

<img width="1206" height="679" alt="Pasted image 20260514005037" src="https://github.com/user-attachments/assets/9bd8c6d0-4ee6-4542-bb52-cb95207dbb64" />

With the `whoami /groups` command, you can check if the user your currently on belongs to this group.

<img width="1225" height="300" alt="Pasted image 20260514005454" src="https://github.com/user-attachments/assets/3e8dd08b-7c53-41bf-8bc4-23930e650434" />

An alternative way to do all this, would be to simply execute the following powershell command. Keep in mind that the command below is considering the username admin (that shows in the end of the command). You'd have to replace it with your own.
```
Add-ADGroupMember -Identity "Domain Admins" -Members admin
```

## Editing Group Policies
To get telemetry from Windows, Group Policies need to be edited. On the Server Manager GUI, on the top-right, click on "Tools" and then "Group Policy Management".

Next, in "domain.local", inside "Group Policy Objects", right click on "Default Domain Controllers Policy" and click "Edit". Then, the Group Policy Management Editor window opens, on "Computer Configuration" -> "Windows Settings" -> "Security Settings" -> "Advanced Audit Policy Configuration" -> "Audit Policies" we can find what we need to setup.

<img width="1111" height="668" alt="Pasted image 20260514010107" src="https://github.com/user-attachments/assets/bf20b8f9-e195-4aca-82a1-860935e29488" />

### Domain Controller Credential Validation
Starting by "Account Logon", double click it and setup "Audit Credential Validation" to success + failure. Press "Apply" and "OK".

<img width="1164" height="636" alt="Pasted image 20260514010518" src="https://github.com/user-attachments/assets/88afe544-b8c6-4e29-8278-3d3c4d63ff29" />

Next, do the same for "Audit Kerberos Authentication Service" and "Audit Kerberos Service Ticket Operations".

<img width="430" height="87" alt="Pasted image 20260514010646" src="https://github.com/user-attachments/assets/4d906ab8-8fee-453e-a90c-419a6943ee7e" />

### Account Management
Next, on the Audit Policies is Account Management tab. Here we'll toggle the telemetry related to user account management, which is very important to detect persistence techniques and also security group management which is not only useful for persistence techniques but also privilege escalation.

In this tab, setup "Audit Security Group Management" and "Audit User Account Management" to Success and Failure. After this we'll be able to trigger events such as [Windows Event ID 4720](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4720) ( User Account was created ) and [Windows Event ID 4728](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4728) ( A member was added to a security-enabled global group ), among many other events.

<img width="801" height="662" alt="image" src="https://github.com/user-attachments/assets/bf970b19-6a1f-4dac-87e8-d3d89afc393b" />

### Logon/Logoff

