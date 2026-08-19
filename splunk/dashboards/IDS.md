# IDS Dashboard

This dashboard is dedicated to the IDS used in this lab, Suricata and its alerts. This dashboard will be dedicated to port scans, since it was the major focus for the usage of this IDS.  
Make sure to make use of the following [page](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/abstract) for reference on the Splunk commands.

## Overview
<img width="1866" height="394" alt="Pasted image 20260707103909" src="https://github.com/user-attachments/assets/89196bd1-354e-44f1-841a-853986841d9f" />
<img width="1862" height="392" alt="Pasted image 20260707103922" src="https://github.com/user-attachments/assets/f9bafa58-e0cf-4b99-bf07-3673e7214f80" />
<img width="1861" height="577" alt="Pasted image 20260707103939" src="https://github.com/user-attachments/assets/9e3e8987-7f94-4f27-84ac-39443f2a363b" />
<img width="1865" height="495" alt="Pasted image 20260707104014" src="https://github.com/user-attachments/assets/af91a604-9e52-4f08-91e5-420a45a87307" />


## In-Depth Analysis
### Total Alerts
The dashboard starts with a counter of the total amount of alerts. This servers as a quick visual indicator that can help in understanding if there is something wrong.

<img width="733" height="374" alt="Pasted image 20260707104134" src="https://github.com/user-attachments/assets/8fbe3155-afd2-4d44-8af7-d261d755fc9a" />

```
index=suricata source="/opt/suricata/alerts.log" NOT "SURICATA STREAM"
| where dest_port!=9997
| stats count as "Total Alerts"
```
**Query analysis**:
 - The first line of the query consists on filtering for suricata's splunk index and the suricata alert log file. The `NOT "SURICATA STREAM"` part, is meant to filter out the suricata stream alerts that are extremely noisy and often a result of benign traffic, resulting in a lot of false positives;
 - The second line filters out any events related to port 9997, since this is the port that is setup for Splunk's receiving and would result in a lot of false positives;
 - The last line simply makes use of the `stats count` command to get the radical counter.


### Total Alerts Timechart
Next to the counter, there is a timechart that displays the amount of alerts in intervals of 15 minutes.

<img width="737" height="381" alt="Pasted image 20260707104242" src="https://github.com/user-attachments/assets/f535ea00-74e7-4a27-9c2b-040d072dae98" />

```
index=suricata source="/opt/suricata/alerts.log" NOT "SURICATA STREAM"
| where dest_port!=9997
| timechart span=15m count
```
**Query analysis:**
 - This query follows the same logic as the previous one. The only difference being the way we present the data. Instead of using the `stats count` command to get the counter, the `timechart` command was used to get a timechart. The arguments `span=15m count` were added to count the events in 15 minute intervals.


### Recent Alerts
In this table, we present the same information as the previous charts, but in a more specific way. This time, in the form of a table with columns that further explain each alert.

<img width="1494" height="382" alt="Pasted image 20260707104420" src="https://github.com/user-attachments/assets/4661a650-107a-476a-80e6-a6da94f35cc4" />

```
index=suricata source="/opt/suricata/alerts.log" NOT "SURICATA STREAM"
| where dest_port!=9997
| table _time signature priority protocol src_ip src_port dest_ip dest_port
| sort -_time
```
**Query analysis:**
 - The first two lines of the query are exactly the same as the previous ones, gathering exactly the same information;
 - The third line is where the information is presented in a different way, by using the `table` command. Here a table was built, with the time, signature, priority, protocol, source ip, source port, destination ip and destination port columns;
 - The last line simply makes use of the `sort` command to sort the events in order to show the most recent first.


### Top Signatures
This table shows the suricata alert signatures that were triggered by a possible attack. In the example below, it is possible to see that the signature with the highest amount of alert counts, by far, is the custom signature that refers to port scans.

<img width="1492" height="314" alt="Pasted image 20260707104536" src="https://github.com/user-attachments/assets/fe4f50f4-5b0a-4768-928c-96e00eaea84d" />

```
index=suricata NOT "SURICATA STREAM"
| rex "\[\d+:\d+:\d+:\]\s(?<signature>.*?)\s\[\*\*\]"
| top signature
| fields - _tc
```
**Query analysis:**
 - The first line of the query follows the same logic of the previous queries. The only difference is that we are not filtering for the `alerts.log` file;
 - Since we are facing logs in raw format, the `rex` command was used to parse for the signature's name in the logs and store that value under the variable "signature";
 - Then, the [top](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/top) command was used, which tells Splunk to find the most common values of the "signature" variable and count how often each one of them occur;
 - The last line is essentially an output cleanup to remove an unnecessary field generated by the `top` command. Using the [fields](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/fields) command, the `_tc` field was removed from the table.


### Top Scanning Hosts
This table is meant to showcase the hosts that are the source of port scans.

<img width="737" height="303" alt="Pasted image 20260707104734" src="https://github.com/user-attachments/assets/19b5d3d8-e595-4e7d-b65c-62c97c54797d" />

```
index=suricata "POSSIBLE PORT SCAN"
| rex "\{TCP\}\s(?<src_ip>\d+.\d+\.\d+.\d+)"
| where dest_port!=9997
| stats count by src_ip
| sort -count
```
**Query analysis:**
 - The first line of the query consists on a filter for the custom suricata signature ("POSSIBLE PORT SCAN"), in order to get the logs from the related alerts;
 - Then, on the second line, using the `rex` command, the value for the source ip was parsed out of the log and stored on the variable "src_ip";
 - The third line consists on the already mentioned filter to exclude logs which target the port 9997, which is the port setup for Splunk's receiving;
 - The fourth line consists on using the `stats count` command to count the amount of obtained events by source ip;
 - The last line sorts the table to show the count on the table in descending order, showing the ip with the highest amount of events first.


### Targeted Hosts
This table is meant to show the machines that were targets of port scanning.

<img width="733" height="291" alt="Pasted image 20260707104956" src="https://github.com/user-attachments/assets/75e1db11-a7d7-4608-82e0-abcd3d02d6b8" />

```
index=suricata "POSSIBLE PORT SCAN"
| rex "->\s(?<dst_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by dst_ip
| where count > 5
| sort -count
```
**Query analysis:**
 - This query follows a very similar logic to the previous one. The first line is exactly the same, filtering for the custom suricata signature;
 - Then, once again using the `rex` command, the value for the destination ip was parsed out of the log and stored on the variable "dst_ip";
 - Then, using the `stats count` command, the amount of events by destination ip was counted;
 - Then, on the fourth line of the query, the command `where` was used to filter out events that have a count lower than 5, to filter false positives out, excluding regular benign communication;
 - Finally the `sort` command was used to sort the events in descending order, similarly to the previous table;
 - Looking back at this query, the filter `where dest_port!=9997` should be added to filter out the communication of Splunk's TCP receiver. This would result in a lower amount of events in the table, in this case, for the ip 10.20.20.5, which is the linux server hosting Splunk.


### Most Targeted Ports
This table is meant to showcase the most target ports on port scans. This can be a helpful indicator that can indicate which ports and services may be in need of hardening or extra attention.

<img width="1490" height="274" alt="Pasted image 20260707105133" src="https://github.com/user-attachments/assets/f36a67ac-f318-48f0-9f36-e1ae977ff466" />

```
index=suricata "POSSIBLE PORT SCAN"
| rex "->\s(?dst_ip>\d+\.\d+\.\d+\.\d+):(?<dst_port>\d+)"
| stats count by dst_port
| where dst_port!=9997
| sort -count
```
**Query analysis:**
 - The first line is the same as the previous query;
 - The second line, besides parsing from the log the value for destination ip, this time it also parses the value for the destination port, storing this value on the variable "dst_port";
 - Then, using the `stats count` command, the amount of events by destination port was counted;
 - The following line filters out events for the destination port 9997, which as previously mentioned, is Splunk's receiving TCP port;
 - Finally, using the `sort` command, the events were sorted in descending order, shower the events with the highest count first;
 - By looking at the table it is also possible to observe that there is an unusually bigger count for the port 443 when compared to other ports. This may be because of regular benign traffic when making use of the applications. After further studying this possibility, filtering out the traffic for port 443 should be evaluated to clean up the output.


### Scan Activity Timeline
This timechart is meant to showcase a timeline for port scanning, helping with chronological visualization of the events.

<img width="1851" height="296" alt="Pasted image 20260707105337" src="https://github.com/user-attachments/assets/838e4040-3bb8-46e9-a159-90f67998dde5" />

```
index=suricata "POSSIBLE PORT SCAN"
| where dest_port!=9997
| timechart count
```
**Query analysis:**
 - The first line is the same as the previous queries;
 - The second line consists on the filter to exclude the traffic headed to port 9997 which, as previously explained, is benign;
 - The third line simply makes use of the `timechart` command to build a timechart with the count of triggered alerts originated from the custom signature `POSSIBLE PORT SCAN`.
