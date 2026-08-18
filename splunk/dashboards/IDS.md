# IDS Dashboard

This dashboard is dedicated to the IDS used in this lab, Suricata and its alerts. Make sure to make use of the following [page](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/abstract) for reference on the Splunk commands.

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
