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
The first table of the Windows dashboard, showcases the execution of suspicious processes. By keeping an eye on commonly used processes in attacks, associated command lines and their parent images, it is possible to quickly detect any anomaly.

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
