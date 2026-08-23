# Introduction
I've decided to include some Windows machines on this project to be able to work with windows telemetry and active directory. These machines come in the form of a Windows Server 2019, which will act as a Domain Controller, and a Windows 11 endpoint which will simulate a simple user endpoint.

# Contents
1. [Setting up the Windows Server VM](#Setting-up-the-Windows-Server-VM)
   - [Setting up the network connection](#Setting-up-the-network-connection)
   - [Adding a new user to the Domain Admin group](#Adding-a-new-user-to-the-Domain-Admin-group)
   - [Editing Group Policies](#Editing-Group-Policies)
     - [Domain Controller Credential Validation](#Domain-Controller-Credential-Validation)
     - [Account Management](#Account-Management)
     - [Logon/Logoff](#Logon/Logoff)
     - [DS Access](#DS-Access)
     - [Policy Change](#Policy-Change)
     - [Privilege Use](#Privilege-Use)
     - [System](#System)
     - [Applying the Group Policy changes](#Applying-the-Group-Policy-changes)
   - [Toggle PowerShell Logging](#Toggle-PowerShell-Logging)
   - [Installing Splunk Universal Forwarder](#Installing-Splunk-Universal-Forwarder)
   - [Setting up Sysmon](#Setting-up-Sysmon)
2. [Setting up the Windows 11 endpoint VM](#Setting-up-the-Windows-11-endpoint-VM)
   - [Include the machine on the AD domain](#Include-the-machine-on-the-AD-domain)
   - [Network connection setup](#Network-connection-setup)
   - [Splunk Universal Forwarder](#Splunk-Universal-Forwarder)
   - [Sysmon Setup](#Sysmon-Setup)
   - [PowerShell Logging](#PowerShell-Logging)

# Setting up the Windows Server VM
Before booting the machine, in virtualbox, this machine will only be working with one network adapter, connected to our internal network.

<img width="388" height="49" alt="image" src="https://github.com/user-attachments/assets/581b4312-3213-4e43-a5c5-b2c2108f9cac" />

After going through the installation, its a good idea to update the OS with the latest updates before proceeding.

To change the name of the machine, the following powershell command can be used:
```
Rename-Computer -NewName "your-name" -Restart
```

[Back to top](#Contents)

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

[Back to top](#Contents)

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

[Back to top](#Contents)

## Editing Group Policies
To get telemetry from Windows, Group Policies need to be edited. On the Server Manager GUI, on the top-right, click on "Tools" and then "Group Policy Management".

Next, in "domain.local", inside "Group Policy Objects", right click on "Default Domain Controllers Policy" and click "Edit". Then, the Group Policy Management Editor window opens, on "Computer Configuration" -> "Windows Settings" -> "Security Settings" -> "Advanced Audit Policy Configuration" -> "Audit Policies" we can find what we need to setup.

<img width="1111" height="668" alt="Pasted image 20260514010107" src="https://github.com/user-attachments/assets/bf20b8f9-e195-4aca-82a1-860935e29488" />

[Back to top](#Contents)

### Domain Controller Credential Validation
Starting by "Account Logon", double click it and setup "Audit Credential Validation" to success + failure. Press "Apply" and "OK".

<img width="1164" height="636" alt="Pasted image 20260514010518" src="https://github.com/user-attachments/assets/88afe544-b8c6-4e29-8278-3d3c4d63ff29" />

Next, do the same for "Audit Kerberos Authentication Service" and "Audit Kerberos Service Ticket Operations".

<img width="430" height="87" alt="Pasted image 20260514010646" src="https://github.com/user-attachments/assets/4d906ab8-8fee-453e-a90c-419a6943ee7e" />

[Back to top](#Contents)

### Account Management
Next, on the Audit Policies is Account Management category. Here we'll toggle the telemetry related to user account management, which is very important to detect persistence techniques and also security group management which is not only useful for persistence techniques but also privilege escalation.

In this tab, setup "Audit Security Group Management" and "Audit User Account Management" to Success and Failure. After this we'll be able to trigger events such as [Windows Event ID 4720](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4720) ( User Account was created ) and [Windows Event ID 4728](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4728) ( A member was added to a security-enabled global group ), among many other events.

<img width="801" height="662" alt="image" src="https://github.com/user-attachments/assets/bf970b19-6a1f-4dac-87e8-d3d89afc393b" />

[Back to top](#Contents)

### Logon/Logoff
The Logon/Logoff category is responsible for the telemetry related to, as the name suggests, logon and logoff events. 

On this tab, we'll setup the subcategories "Audit Logoff", "Audit Logon" and "Audit Special Logon" for Success and Failure. After setting this tab up, events such as [Windows Event ID 4624](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4624) ( An account was successfully logged on ) and [Windows Event ID 4624](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4625) ( An account was successfully logged off ) will be triggered.

<img width="454" height="269" alt="image" src="https://github.com/user-attachments/assets/2c508142-d9f1-4c15-8aee-f9d29a718e05" />

[Back to top](#Contents)

### DS Access
This category monitors changes in directory service. Any changes made to objects in Active Directory will generate an alert.

To setup this tab, the subcategories "Audit Directory Service Access" and "Audit Directory service Changes" need to be setup to Success and Failure. After setting this tab up, events such as [Windows Event ID 4662](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4662) ( An operation was performed on an object ) or [Windows Event ID 5137](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=5137) ( A directory service object was created ) will be triggered.

<img width="424" height="115" alt="image" src="https://github.com/user-attachments/assets/d572eb6c-3f6a-473c-bc91-8424a626a9e4" />

[Back to top](#Contents)

### Policy Change
The Policy Change category detects changes to the security policies that govern Windows machines. This can be valuable to a SOC since an attacker who gets administrative privileges may try to weaken auditing or security controls to make their activity harder to detect.

For this category, we'll setup the subcategories "Audit Policy Change" and "Audit Authentication Policy Change" to Success and Failure. After setting this up, events such as [Windows Event ID 4719](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4719) ( System audit policy was changed ) and [Windows Event ID 4739](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4739) ( Domain Policy was changed ).

<img width="421" height="150" alt="image" src="https://github.com/user-attachments/assets/3e0f41b9-3fc3-4397-b098-97a13cea62e3" />

[Back to top](#Contents)

### Privilege Use
This category monitors the use of high-impact privileges such as [SeImpersonatePrivilege](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/seimpersonateprivilege-secreateglobalprivilege), among many others, which can be used by attackers to escalate privileges or bypass security controls.

For this category, we'll set the subcategory "Audit Sensitive Privilege Use" to Success and Failure. Once setup, this will trigger events such as [Windows Event ID 4673](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4673) ( A privileged service was called ) and [Windows Event ID 4739](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4674) ( An operation was attempted on a privileged object ).

<img width="422" height="99" alt="image" src="https://github.com/user-attachments/assets/88a9bda6-22cc-4c5f-8479-caa2ccd72d76" />

[Back to top](#Contents)

### System
This category monitors changes that can be described as system changes with potential security implications. On this category, we'll monitor for changes to the system's security state and loading security packages/extensions and installing services.

For this category, we'll setup the "Audit Security State Change" and "Audit Security System Extension" subcategories to Success and Failure. Once setup, events such as [Windows Event ID 4608](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4608) ( Windows is starting up ) and [Windows Event ID 4697](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4697) ( A service was installed on the system ).

<img width="422" height="128" alt="image" src="https://github.com/user-attachments/assets/c98106c3-a781-4e46-b773-3b5e5443cb69" />

[Back to top](#Contents)

### Applying the Group Policy changes
On a powershell terminal with admin rights, run the command `gpupdate /force`. To check if the policies were applied correctly, run the command `auditpol /get /category:*`.

[Back to top](#Contents)

## Toggle PowerShell Logging
To toggle powershell logging, on the Server Manager app, click on "Tools" and then "Group Policy Management". Inside `domain.local`, go to "Group Policy Objects" and right click on "Default Domain Controllers Policy" and then click "Edit". On the new windows, go to "Computer Configuration" -> "Policies" -> "Administrative Templates" -> "Windows Components" -> "Windows Powershell".

<img width="1151" height="647" alt="Pasted image 20260514224903" src="https://github.com/user-attachments/assets/f8d51344-7eca-4fcc-aef4-65b96f18e010" />

Double click on "Turn on Module Logging", pick "Enabled" and then click on "Show..." and add a wildcard with `*`. Press "OK" and "Apply".

<img width="1132" height="643" alt="Pasted image 20260514225045" src="https://github.com/user-attachments/assets/87d3fc14-8edd-40c6-927e-2c080fe91536" />

Next, Enable "Script Block Logging" and "Powershell Transcription". For Powershell Transcription, add the directory "C:\PSTranscripts". Powershell transcription is essentially a session logging for Powershell that creates human-readable text files containing the commands executed in Powershell. In this case, the output of this logging will be sent to the directory `C:\PSTranscripts`.

<img width="1083" height="741" alt="Pasted image 20260514225303" src="https://github.com/user-attachments/assets/ddb71e92-8613-464c-96bf-b09ad26cb398" />

Once done, apply the changes on a powershell terminal by running the command `gpupdate /force`.

[Back to top](#Contents)

## Installing Splunk Universal Forwarder
Download the Splunk Universal Forwarder setup from the [official Splunk website](https://www.splunk.com/en_us/download/universal-forwarder.html). The universal forwarder, is the tool that collects data remotely from various sources and forwards it to Splunk.

There will be a time when you'll be asked to setup the receiving indexer, here you'll have to setup with the IP of the machine that is hosting the Splunk service, in this case the Linux Server VM, and the port that is setup for that purpose, in this case port 9997.

Once the setup is done, head to the directory `C:\Program Files\SplunkUniversalForwarder\etc\system\local`. Here, create a file called "inputs.conf", which will be the file that will tell the universal forwarder which logs to collect.

<img width="1918" height="916" alt="image" src="https://github.com/user-attachments/assets/c129fb3b-a6ed-41fc-9013-6c8218652f66" />

The picture above already has lines setup for sysmon logs, which will be setting up next. Also, notice the presence of indexes. These are a nice way to organize logs in Splunk, allowing you to separate logs in an organized manner instead of having them stored all together. Writing the indexes in this file tells the universal forwarder where it has to send each of these logs.

Once you're done, open a powershell terminal and head to the `C:\Program Files\SplunkUniversalForwarder\bin` directory and restart Splunk with the `.\splunk restart` command.

<img width="1095" height="339" alt="Pasted image 20260515130146" src="https://github.com/user-attachments/assets/62f6e183-1137-4dfc-89bd-9cee986641b8" />

[Back to top](#Contents)

## Setting up Sysmon
To begin, download Sysmon from the official [site](https://download.sysinternals.com/files/Sysmon.zip). Then, it is also important to download a sysmon config file, the best one to begin with will be [SwiftOnSecurity's sysmon config file](https://github.com/swiftonsecurity/sysmon-config).

With both the tools downloaded on the same directory and the initial Sysmon zip file decompressed, run the command `.\Sysmon64.exe -accepteula -i sysmonconfig.xml`

<img width="620" height="291" alt="Pasted image 20260515162925" src="https://github.com/user-attachments/assets/9b1b9300-b46f-44ed-b1f2-c56011b4319f" />

To make sure that it was done correctly, run the command `Get-Service Sysmon64`.

<img width="304" height="89" alt="Pasted image 20260515163033" src="https://github.com/user-attachments/assets/f99e11fd-3b78-493d-a91f-8adc3012f557" />

If you're following this guide in a sequence, you should have your Splunk Universal Forwarder `inputs.conf` file setup to include these logs. If not, head to `C:\Program Files\SplunkUniversalForwarder\etc\system\local` and add the following line.

<img width="436" height="70" alt="image" src="https://github.com/user-attachments/assets/d369f7f6-04b9-4ad3-a0cc-efc957e03547" />

After every change to the `inputs.conf` file, open a powershell terminal and head to the `C:\Program Files\SplunkUniversalForwarder\bin` directory and restart Splunk with the `.\splunk restart` command.

Next, you can head to Splunk and check for these logs with the command `index=sysmon sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational"`

<img width="1908" height="901" alt="Pasted image 20260515163523" src="https://github.com/user-attachments/assets/8d02c4bd-bb0f-4ce0-9f4c-b7ecc81ecb1b" />

You can also generate a bunch of [Sysmon Event ID 1](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90001) logs by opening a powershell terminal and executing commands like `notepad`, `whoami`, `ipconfig` or a simple `ping 8.8.8.8`.

<img width="1889" height="876" alt="Pasted image 20260515164052" src="https://github.com/user-attachments/assets/db5725cd-98a1-4593-b093-251e5859540d" />

[Back to top](#Contents)

# Setting up the Windows 11 endpoint VM
Considering that most of the setup for this machine will be the same as the Windows Server VM, given that they are both Windows machines, this section will be more streamlined than the previous one, to avoid repetition. Please refer to the previous section in case something isn't clear enough, I'll also make references to the previous section.

Just like the Windows Server VM, before booting the machine, in virtualbox, this machine will only be working with one network adapter, connected to our internal network.

<img width="388" height="49" alt="image" src="https://github.com/user-attachments/assets/581b4312-3213-4e43-a5c5-b2c2108f9cac" />

As mentioned for the Windows Server VM, you can start by renaming the machine with the command `Rename-Computer -NewName "yourname" -Restart` and also, the internet adapter with the command `Rename-NetAdapter -Name "oldname" -NewName "newname"

<img width="1005" height="227" alt="Pasted image 20260517180221" src="https://github.com/user-attachments/assets/823161de-a27f-4d39-b6ac-eb5410441a6a" />

[Back to top](#Contents)

## Include the machine on the AD domain
Before proceeding, we have to include this machine on the AD domain where our DC resides. Search for "View Advanced System Settings", click on it, then head to "Computer Name", click "Change..." and on the "Member of Domain:" section, write the name of DC's domain and press OK.

<img width="885" height="811" alt="image" src="https://github.com/user-attachments/assets/34b4cfab-e8ca-4f88-93a4-c6b3b6f93190" />

Next, restart the machine and login as a user that is inside that domain, with the domain name before the username as in `domainname/username` in the login panel.

From the DC machine, you should be able to see your newly added Windows 11 machine.

<img width="756" height="532" alt="Pasted image 20260517183030" src="https://github.com/user-attachments/assets/8767c10c-4865-4c07-a8b0-26ca97559d1f" />

[Back to top](#Contents)

## Network connection setup
Since we don't have access to the "Server Manager" app from this machine, we'll do the setup from the command line. 

To setup the machine's IP address, considering that you want to setup your machine with the IP address of 10.10.20.20 for the interface "LABNET" run `New-NetIPAddress -InterfaceAlias "LABNET" -IPAddress 10.10.20.20 -PrefixLength 24`.

Then, to setup the DNS server, we'll want to refer to our DC for this. Considering that our DC has the IP of 10.10.20.10, run the command `Set-DnsClientServerAddress -InterfaceAlias "LABNET" -ServerAddresses 10.10.20.10`. Before doing this its a good idea to try and ping the DC machine to see if the machines can see each other and, in case they don't see each other, troubleshoot and solve it before proceeding.

Next, run the command `nslookup domain.local` and you should see the DC's IP address on the "Address" field. For the example below, the DC's IP was 10.10.10.10, at the time.

<img width="466" height="236" alt="Pasted image 20260517180944" src="https://github.com/user-attachments/assets/f0b3f3b9-5703-4a29-bf0d-c6a27262e040" />

[Back to top](#Contents)

## Splunk Universal Forwarder
To install and setup Splunk Universal Forwarder, with the exception of the `inputs.conf` file, the procedure is exactly the same as the [one from the Windows Server VM](#Installing-Splunk-Universal-Forwarder), so please refer to it.

Make sure to setup the `input.conf` file as shown on the picture below.

<img width="509" height="517" alt="image" src="https://github.com/user-attachments/assets/2070ab95-0963-430e-954f-ea4df1cb70c6" />

[Back to top](#Contents)

## Sysmon Setup
To install and setup Sysmon, the procedure is exactly the same as the [one from the Windows Server VM](#Setting-up-Sysmon), so please refer to it.

[Back to top](#Contents)

## PowerShell Logging
To setup PowerShell Logging, the procedure is exactly the same as the [one from the Windows Server VM](#Toggle-PowerShell-Logging), so please refer to it.

[Back to top](#Contents)

## Activate RDP
For the attack simulation that I'll do, I chose to use RDP as an attack vector. To it to be possible, RDP must be turned on. This is only for demonstration purposes, for general use it is advised to keep RDP turned off as its very vulnerable to attacks and a very common attack vector.

In "Settings", head to "System" and then look for "Remote Desktop". Select it and when inside simply turn On Remote Desktop.

<img width="706" height="131" alt="image" src="https://github.com/user-attachments/assets/ba3cb531-785c-461f-9fbd-ed6eb736f937" />
