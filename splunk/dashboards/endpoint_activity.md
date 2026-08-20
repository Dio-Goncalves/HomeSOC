# Endpoint Activity Dashboard

This dashboard is made of two different tabs:
 - The [Windows tab](#Windows-tab) which will focus on the windows machines;
 - The [Linux tab](#Linux-tab) which will focus on the linux machines.

For reference on the Splunk commands, please refer to this [page](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/abstract). For reference on Windows Event IDs, please refer to this [page](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/).

## Windows tab
### Overview
<img width="1872" height="614" alt="Pasted image 20260708095906" src="https://github.com/user-attachments/assets/5854ae3e-d64b-4093-9592-7c04f8971e3f" />
<img width="1863" height="573" alt="Pasted image 20260708095928" src="https://github.com/user-attachments/assets/5e3e11b4-dd6c-4dc3-9a0d-a437cc2bf9be" />
<img width="1858" height="469" alt="Pasted image 20260708095954" src="https://github.com/user-attachments/assets/348e036c-5c6c-449d-ac52-d10a332f2a3a" />
<img width="1854" height="268" alt="Pasted image 20260708100005" src="https://github.com/user-attachments/assets/23b660cf-cf05-40b9-a9ba-94ec3ccea25f" />

### In-Depth Analysis
#### Suspicious Process Execution
The first table of the Windows dashboard, showcases the execution of suspicious processes. By keeping an eye on commonly used utilities in attacks, associated command lines and their parent images, it is possible to quickly detect any anomaly.

<img width="1851" height="379" alt="Pasted image 20260708100031" src="https://github.com/user-attachments/assets/741ddd11-6e07-4bbd-b700-dfceebefc923" />

```
index=* EventCode=1 NOT "SplunkUniversalForwarder"
| eval Process=lower(Image)
| where like(Process,"%powershell%")
OR like(Process,"%cmd.exe%")
OR like(Process,"%wmic%")
OR like(Process,"%certutil%")
OR like(Process,"%mshta%")
OR like(Process,"%rundll32%")
OR like(Process,"%regsvr32%")
OR like(Process,"%bitsadmin%")
OR like(Process,"%psexec%")
| eval User=mvindex(User,-1)
| where ParentImage!="C:\Windows\System32\*"
| table _time host User Image CommandLine ParentImage
| sort -_time
```
**Query analysis:**
 - The first line of the query searches for [Sysmon Event ID 1](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90001) events, referring to process creation. It also exculdes events related to Splunk Universal Forwarder, since this is benign traffic and would simply generate unnecessary noise;
 - Using, `eval` a normalized "Process" variable was created;
 - Next, using the created normalized variable, the logs were filtered for the suspicious processes, using `Process` as a temporary field for filtering;
 - Then, using the `eval` command, the last value of the `User` field is parsed and stored under the variable "User";
 - Using the `where` command, events related to the Windows System 32 path were filtered, since these are related to the SYSTEM account and are usually benign, generating unnecessary noise;
 - Then, using the `table` command, a table was built with the time, host, User, Image, CommandLine and ParentImage columns;
 - Finally the table was sorted by time, showing the most recent events first.


#### Scheduled Task Creation
A common way for attackers to gain persistence is via scheduled tasks. These can fly under the radar rather effectively and are a good way of getting a backdoor to the victim's machine. This table monitors for the creation of scheduled tasks, showing the related images, command line, target filename and, of course, timestamp and host.

<img width="1494" height="303" alt="Pasted image 20260708100101" src="https://github.com/user-attachments/assets/65e44584-c42c-4676-b453-856b5d95a4bf" />

```
(index=sysmon EventCode=1 Image="*schtasks.exe*") OR (index=sysmon EventCode=11 TargetFilename="C:\\Windows\\System32\\Tasks\\*")
| table _time host Image CommandLine TargetFilename
| sort -_time
```
**Query analysis:**
 - The first line filters for events that either refer to process creation ( [Sysmon Event ID 1](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90001) ) using the commmand line interface for the task scheduler, `schtasks.exe` or events that refer to file creation ( [Sysmon Event ID 11](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90011) ), monitoring the directory where files related to scheduled tasks are stored;
 - Then, using the `table` command, a table with the time, host Image, CommandLine and TargetFilename columns is built;
 - Finally, the table was sorted by time, showing the most recent events first.


#### Powershell Activity
The following table monitors for powershell activity. By displaying the related images to the powershell process, together with the user and host, it can be helpful in evaluating if the powershell process is malicious. Looking back, even though this table can be useful, its rendered rather useless by the table that will be exposed next. In order to reduce visual clutter, it wouldn't be a bad idea to see this table's presence as optional.

<img width="1494" height="263" alt="Pasted image 20260708100140" src="https://github.com/user-attachments/assets/a72f779a-eb0b-46e5-b544-d44db9727b70" />

```
index=sysmon EventCode=1 Image="*powershell.exe*"
| where Image!="C:\Program Files\SplunkUniversalForwarder\bin\splunk-powershell.exe"
| eval User=mvindex(User,-1)
| table _time host User Image CommandLine ParentImage
```
**Query analysis:**
 - This query starts by filtering, once again, for [Sysmon Event ID 1](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90001), related to process creation. It also searches for Images that make reference to `powershell.exe`;
 - Next, images related to the Splunk Universal Forwarder where filtered out to reduce noise;
 - Then, the last value of the User field was parsed and stored under the variable "User";
 - Finally the table was built with the time, User, Image, CommandLine and ParentImage columns.


#### Suspicious Powershell Commands
This table gives a different perspective on powershell activity when compared to the last table. This table actually shows the executed commands. As an improvement, it could be useful to implement the `User` field on the table.

<img width="1498" height="443" alt="Pasted image 20260708100204" src="https://github.com/user-attachments/assets/ca68cdc0-7570-4338-b864-d010625e5939" />

```
index=powershell source="WinEventLog:Microsoft-Windows-Powershell/Operational" EventCode=4104
(Message="*Invoke-Expression*"
OR Message="*DownloadString*"
OR Message="*EncodedCommand*"
OR Message="*Net.WebClient*"
OR Message="*Invoke-Mimikatz*"
OR Message="*iex*"
OR Message="*net*"
OR Message="*FromBase64String*"
| table _time ComputerName Message
| sort -_time
```
**Query analysis:**
 - The query starts by filtering for the powershell index and the Windows' powershell operational source of data. Notice that there is also a filter for Windows Event ID 4104, this event refers to Powershell Script Block Logging and it captures the text of scrips executed via powershell;
 - Then, the `Message` field is filtered for the most common powershell utilities used in attacks;
 - Finally a table is built with the time, ComputerName and Message columns, and sorted by time to show the most recent events first.


#### Parent and Child Process Relationships
Similar to the Suspicious Process Execution table, this table monitors for the relationships between parent and child processes, monitoring the parent images that are usually common vectors of attack. It also displays executed commands. The difference in this table is that it focuses on the the parent image, rather than the child image like the first one does.

<img width="1488" height="460" alt="Pasted image 20260708100236" src="https://github.com/user-attachments/assets/b5a5dcc5-f6b4-437e-b65b-0575ad7850db" />

```
index=* source="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
| search ParentImage="*\\winword.exe"
OR ParentImage="*\\excel.exe"
OR ParentImage="*\\outlook.exe"
OR ParentImage="*\\powershell.exe"
OR ParentImage="*\\cmd.exe"
| eval User=mvindex(User,-1)
| table _time host User ParentImage Image CommandLine
| sort -_time
```
**Query analysis:**
 - The query starts by filtering for the sysmon operational source and [Sysmon Event ID 1](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90001), related to process creation;
 - Then, using the [search](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/search) command, the `ParentImage` field is filtered to look for applications that are common attack vectors, like powershell, cmd or even microsoft office tools, via macros;
 - Then, the final value of the `User` field is parsed and stored on the variable "User";
 - Finally, using the `table` command, a table with the time, host, User, ParentImage, Image and CommandLine columns is built. The table is sorted by time to show the most recent events first.


#### Registry Persistence
This table monitors registry key activity, which is a very common way of gaining persistence on a machine. With this a program can configure itself to run automatically when Windows or a user starts.

<img width="1496" height="292" alt="Pasted image 20260708100258" src="https://github.com/user-attachments/assets/9d5a4960-ae20-452d-b8df-13c29e047c87" />

```
index=* source="WinEventLog:Microsoft-Windows-Sysmon/Operational" (EventCode=12 OR EventCode=13) (TargetObject="*\\Run\\*" OR TargetObject="*\\RunOnce\\*" OR TargetObject="*\\StartupApproved*")
| eval User=mvindex(User,-1)
| table _time host User TargetObject Details Image
| sort -_time
```
**Query analysis:**
 - The query starts by searching across various indexes but only for events coming from the Sysmon Operational log. Then, a filter is applied for the [Sysmon Event ID 12](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90012) ( Registry object create and delete ) and [Sysmon Event ID 13](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90013) ( Registry value set ). Then, the `TargetObject` field was also filtered for persistence-related registry locations, which are related to programs that execute automatically. The `Run` location refers to automatic execution when the user logs in, the `RunOnce` is similar but it executes something once and then removes the entry and, finally the `StartupApproved` field is associated with Windows' management of startup applications;
 - Then, the final value of the `User` field is parsed and stored under the variable "User";
 - Using the `table` command, a table was built with the time, host, User, TargetObject, Details and Image columns. In this case, the `TargetObject` field refers to the registry value that was affected, the `Details` field to the data involved in the operation and the `Image` field to the process responsible for the registry activity;
 - Finally the table was sorted by time to show the most recent events first.


## Linux tab
### Overview
<img width="1869" height="429" alt="Pasted image 20260708100358" src="https://github.com/user-attachments/assets/eb3798ce-7a37-4571-94c0-7fe5dd888f02" />
<img width="1858" height="384" alt="Pasted image 20260708100409" src="https://github.com/user-attachments/assets/8df133b3-0610-475d-b6b3-7908d6ae2f7a" />
<img width="1855" height="386" alt="Pasted image 20260708100424" src="https://github.com/user-attachments/assets/5e40d4a4-30d6-4ab7-9391-11292c37cb3b" />
<img width="1854" height="264" alt="Pasted image 20260708100435" src="https://github.com/user-attachments/assets/2581e9c6-e006-410a-a770-a029cf126d24" />

### In-Depth Analysis
#### Sensitive Command Execution
This table focuses on monitoring the execution of sensitive commands, commonly used with malicious intents.

<img width="740" height="346" alt="Pasted image 20260708100454" src="https://github.com/user-attachments/assets/4f2ffb0d-09e4-43bb-8d48-d4b9ba8e4a4c" />

```
index=* source="/var/log/auth.log" "sudo:"
| rex field=_raw "COMMAND=(?<Command>.+)$"
| search COMMAND IN (
"*useradd*",
"*adduser*",
"*userdel*",
"*deluser*",
"*passwd*",
"*usermod*",
"*systemctl*",
"*service*",
"*visudo*",
)
| table _time host USER PWD Command
| sort -_time
```
**Query analysis:**
 - The first line of the query filters for the `auth.log` file, together with the expression `sudo:`, looking for logs that refer to command execution using sudo;
 - The next line of the query, makes use of the `rex` command to parse out the executed commands out of the raw logs and store them in the "Command" variable;
 - Next, the `search` command is used once again to make an intermediary search and match the found Command with a list of sensitive linux-based commands. It is possible to see variants of the same command in this list, like `userdel` and `deluser`, that's because there are Linux and Debian machines in this environment and both need to be covered;
 - The next line, using the `table` command, builds a table with the time, host, user, current working directory (pwd) and command columns;
 - Finally the table is sorted by time to show the most recent events first.


#### Sudo Activity
This table monitors for all sudo activity. Any command that that is executed using sudo, show appear in this table. Given the power that the sudo has, it is imperative to monitor its usage since it can be a major problem in case of compromise. Looking back, this table effectively renders the previous one obsolete and one could consider removing the previous table to streamline the dashboard and reduce visual clutter.

<img width="736" height="359" alt="Pasted image 20260708100517" src="https://github.com/user-attachments/assets/756dd1ff-73a4-4c15-bc00-6f50b4fa1f09" />

```
index=* source="/var/log/auth.log" "sudo:"
| rex field=_raw "COMMAND=(?<Command>.+)$"
| where isnotnull(USER)
| table _time host USER PWD Command
| sort -_time
```
**Query analysis:**
 - The logic behind the first two lines of this query is the same as the previous one;
 - The third line filters out events that have null values on the `USER` field;
 - Using the `table` command, the fourth line build a table with the time, host user, current working directory and command columns;
 - Finally, the table is sorted by time to show the most recent events first.


#### Successful SSH Logins
This table is designed to monitor successful SSH login events. Conceptually, this table should be rather noisy with benign traffic, considering that it is normal to have successful login attempts. It would be wise to think about another way to implement this table or simply remove it from the dashboard. It is also debatable that this table should be in the authentication dashboard instead of this one, considering that we are talking about SSH.

<img width="739" height="289" alt="Pasted image 20260708100543" src="https://github.com/user-attachments/assets/cefd396c-e21b-4528-b11b-0d586014f917" />

```
index=* source="/vat/log/auth.log" sshd "Accepted password"
| rex field=_raw "for (?<User>\S+)"
| rex field=_raw "from (?<SourceIP>\S+)"
| table _time host User SourceIP
| sort -_time
```
**Query analysis:**
 - The first line of the query filters for events related to successful SSH logins in the `auth.log` file;
 - The second line parses the username out of the raw logs and stores it under the "User" variable;
 - The third line parses the source IP address out of the raw logs and stores it under the "SourceIP" variable;
 - Next, using the `table` command, a table with the time, host, User and SourceIP columns is built;
 - Finally, the table is sorted by time to show the most recent events first.


#### Failed SSH Logins
This table is the opposite of the previous one, monitoring Failed SSH login events. Very important to monitor for brute-force attacks, since SSH is usually a common vector of this type of attack. Just like the previous table, it should be considered to move this table to the authentication dashboard, considering we are dealing with SSH.

<img width="734" height="365" alt="Pasted image 20260708100602" src="https://github.com/user-attachments/assets/26942851-a1a8-448a-a6a0-3be6ce910df4" />

```
index=* source="/var/log/auth.log" sshd "Failed password"
| rex field=_raw "for (?:invalid user )?(?<User>\S+)"
| rex field=_raw "from\s+(?<SourceIP>\S+)"
| table _time host User SourceIP
| sort -_time
```
**Query analysis:**
 - The first line of the query filters for events related to failed SSH logins in the `auth.log` file;
 - The second line of the query parses the username out of the raw log and stores it in the "User" variable;
 - The third line parses the source IP address out of the raw log and stores it under the "SourceIP" variable;
 - Next, using the `table` command, a table with the time, host, User and SourceIP columns is built;
 - Finally, the table is sorted by time to show the most recent events first.

#### Service Monitoring and Changes
This table monitors for service monitoring and changes, using the `systemctl` command. This can be a very useful command for attackers to monitor and weaken the victim's machine.

<img width="1498" height="372" alt="Pasted image 20260708100743" src="https://github.com/user-attachments/assets/d92075f4-9576-4e31-b020-012007bb055f" />

```
index=linux source="/var/log/auth.log"
| rex field=_raw "sudo:\s+(?<Invoker>\S+)\s+:"
| rex field=_raw "USER=(?<TargetUser>[^;]+)"
| rex field=_raw "COMMAND=(?<Command>.*)"
| search Command="/usr/bin/systemctl*"
| table _time host Invoker TargetUser Command
| sort -_time
```
**Query analysis:**
 - The first line of the query filters for events in the `auth.log` file, in splunk's linux index;
 - The second line makes use of the `rex` command to parse the username that executes the command out of the raw log. This username is then stored under the variable "Invoker";
 - The third line, also using the `rex` command, parses out the user field from the raw logs and stores it under the "TargetUser" variable. This line could be erased from this query since it'll always be root when we execute the `systemctl` command with sudo. Having the "Invoker" column renders this "TargetUser" rather obsolete, it was merely kept to showcase the possibility of parsing this value out of the log;
 - Next, the executed command was parsed out of the raw log and then stored under the variable "Command", also using the `rex` command;
 - Using the `search` command, an intermediary search was made to match the events that were found with the `systemctl` command;
 - Then, a table was built with the time, host, Invoker, TargetUser and Command columns. Once again, the TargetUser column could be kept out of this table;
 - Finally, the table was sorted by time to show the most recent events first.


#### Cron Changes
This table monitors cron changes. This is a very common vector used by attackers to gain persistence on machines, so its imperative to monitor this kind of activity.

<img width="1498" height="372" alt="Pasted image 20260708100743" src="https://github.com/user-attachments/assets/a0498fb3-cb7d-44c6-92a5-9a4c549fe7b8" />

```
index=linux source="/var/log/syslog" crontab
| rex field=_raw "crontab[\d+\]: \((?<User>[^\)]+)\)\s+(?<Action>.*)"
| table _time host User Action
| sort -_time
```
**Query analysis:**
 - The first line filters for the syslog logs, with the added `crontab` expression to filter specifically for the logs related to crontab;
 - Next, using the `rex` command, the username and action are parsed out of the raw logs and stored under the variables "User" and "Action", respectively;
 - Then, using the `table` command, a table with the time, host, User and Action is built;
 - Finally, the table is sorted by time to show the most recent events first.
