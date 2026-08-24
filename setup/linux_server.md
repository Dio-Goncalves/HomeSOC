# Introduction
In order to have some Linux telemetry and to host Splunk, I've decided to include a dedicated Ubuntu 26.04 LTS VM. I've chosen Ubuntu because its stable, lightweight and as good Linux support.

# Contents
1. [Setting up the VM](#Setting-up-the-Windows-Server-VM)
   - [Setting up the network connection](#Setting-up-the-network-connection)
2. [Setting up Splunk](#Setting-up-Splunk)
   - [Access Splunk via browser](#Access-Splunk-via-browser)
   - [Activating Splunk Receiving](#Activating-Splunk-Receiving)

# Setting up the VM
Before booting up the VM, on VirtualBox, setup the network interface to the Internal Network. Also, have around 4096 MB of memory allocated to the machine and 2 processors. For storage, I'd advise around 80 GB just to be safe.

<img width="388" height="49" alt="image" src="https://github.com/user-attachments/assets/581b4312-3213-4e43-a5c5-b2c2108f9cac" />

First things first, change the root password of the VM. With the command `sudo passwd root`, you'll be prompted to insert the new root password. It is a good practice not to use default credentials as it is very unsafe.

Next, it is also not a good practice to use the `root` user for everything. Create a user with sudo rights to setup and manage the machine. Simply run the command `adduser youruser` (replace youruser) with the username you wish. Then, give the administrative privileges with the command `usermod -aG sudo youruser`. The `-a` flag simply appends the group rather than replacing the user's existing groups, and the `-G` flag adds the user to the sudo group. Then, to change to your new user, simply run `su youruser`, insert the password and that's it. Once you're in the new user, to check that you have the administrative rights, run the command `sudo whoami` and you should get "root" as output.

To check the machine name, run the `hostname` command. In case you want to change it, simply run the command `sudo hostnamectl set-hostname newname` ( replace "newname" with the name you want ).

## Setting up the network connection
Before doing anything else on the machine, we have to setup the network interface, in order to have an internet connection. Run the `ip a` command to see the name of the network interface ( enp0s3, enp0s8, ... ).

Then choose the IP address for this machine and run the command with the IP of your choice. In this case, we'll assume the IP `10.20.20.5` for the interface `enp0s8`: `sudo ip addr add 10.20.20.5/24 dev enp0s8`.

Then, run `sudo ip link set enp0s8 up` to bring the interface up, changing its administrative state from "down" to "up".

Next, we need to find ubuntu's startup configuration file for the network interfaces. To do this, run the command `ls /etc/netplan*.yaml`. Once you find the file, simply do a `sudo nano` on it and proceed to set it up as shown below. The setup I'll show below is considering that I have an IP address of 10.20.20.5 for the splunk machine, the default gateway (pfSense machine) has an IP address of 10.20.20.1 and the DNS server (Windows DC) has an IP address of 10.20.20.10.

```
network:
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
      - 10.20.20.5/24
      routes:
        - to: default
          via: 10.20.20.1
      nameservers:
      addresses:
            - 10.20.20.10
```

Save and quit and run the command `sudo netplan apply`.

# Setting up Splunk
Download Splunk Enterprise from the [official website](https://www.splunk.com/en_us/download/splunk-enterprise.html). Create an account of you don't have one and, once you're in, pick the installer with the .tgz extension and copy the wget link.

<img width="1150" height="397" alt="image" src="https://github.com/user-attachments/assets/cf145d80-62d5-44a6-87f4-91797db14b30" />

Then, on the Linux machine, go to the /opt directory which is the directory reserved for add-on application packages, paste the wget command and run it to download the Splunk installer onto the Linux machine.

Decompress the file with `sudo tar -xzf splunk-*.tgz -C /opt` and once decompressed head to the decompressed file's /bin directory.

Once inside the /bin directory, simply run the `./splunk start --accept-license` command and you should be prompted with the initial setup. Beware here you'll be prompted to choose a port to communicate with Splunk, this is what you'll use to access Splunk via browser.

## Enable Splunk to boot on startup
To enable Splunk to automatically boot every time you start the Linux machine, run the command `sudo /opt/splunk/bin/splunk enable boot-start -user youruser`.

## Enable SSL connection with Splunk for HTTPS connection
To enable a safe connection with Splunk via browser, run the command `sudo nano /opt/splunk/etc/system/local/web.conf`.

Add the settings in the picture below

<img width="266" height="53" alt="Pasted image 20260518155415" src="https://github.com/user-attachments/assets/b79d5878-3cb3-435d-9459-7475e69ebb59" />

Restart Splunk with `sudo /opt/splunk/bin/splunk restart`.

## Access Splunk via browser
By now you should be able to access Splunk via browser. To do this it is best to do it from one of the Windows VMs. Depending on how you setup Splunk, you should be able to access it by using it's IP address and the port you've setup on the installation in [Setting up Splunk](#Setting-up-Splunk). Assuming that our Linux machine has an IP address of 10.20.20.5 and we've chosen port 8008 to communicate with Splunk, you should be able to access Splunk with "https://10.20.20.5:8000".

## Activating Splunk Receiving
Once logged into Splunk via browser, it is possible to see a dashboard with a lot of options. Before touching anything, we need to receive logs on Splunk and to achieve this, we need to setup Splunk Receiving. Go to "Settings" -> "Forward and Receiving" -> "Configure Receiving". Once there you can pick any port you want. On the picture below, I've chosen port 9997, as suggested.

<img width="960" height="235" alt="Pasted image 20260515122717" src="https://github.com/user-attachments/assets/efdb975a-436f-4c02-9b0c-7a33ff97d376" />

Once saved, you can go back to the Linux machine and verify if you have that port listening with the command `sudo ss -tulpn | grep 9997`.

<img width="807" height="40" alt="Pasted image 20260515122827" src="https://github.com/user-attachments/assets/2ff1bb19-bc46-4492-a67a-094c78570dbf" />

