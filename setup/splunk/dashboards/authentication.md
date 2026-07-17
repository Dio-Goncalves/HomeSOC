# Authentication Dashboard

This dashboard is made of three different tabs: 
  - The "general" tab which will give an overview over the environment;
  - The "windows" tab which will focus on the windows machines;
  - The "linux" tab which will focus on the linux machines.

Observation: The time filter for this dashboard is set to match the attack ran on the Simulation 1 page.

## General tab
### Overview
<img width="1431" height="870" alt="image" src="https://github.com/user-attachments/assets/de93cfed-e370-4947-9618-c123ad1d3aef" />


### In-Depth Analysis
#### Total Failed Login Attempts Timechart
The dashboard starts with a timechart of total failed logons. This is a cross-OS chart, as its related to all the endpoints in our environment, Linux and Windows.

<img width="918" height="381" alt="Pasted image 20260707111835" src="https://github.com/user-attachments/assets/bc9e2367-6829-4b7b-9548-aaaba106edc7" />  

In this specific case, we can see 2 spikes of failed login attempts right before 10:20 AM and 10:30 AM, and another failed attempt before 10:25 AM.

To get this timechart, we made use of the [Windows Event ID 4625](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4625) (An account failed to log on) and linux auth.log file. Taking this into account and some minor adjustments to build towards the following the following charts and filter out some poorly parsed logs, the following SPL query was built:
```
index=* ((EventCode=4625 OR (source="/var/log/auth.log" "authentication failure"))
| eval User=Case(isnotnull(Account_Name),lower(mvindex(Account_Name,-1)), isnotnull(user), lower(user))
| search User!="win11-client"
| where NOT like(User,"domain/%")
| timechart count
```
**Query Analysis:**  
 - Analyzing the query, its obvious that the first line references what was previously mentioned regarding the Windows Event ID and Linux auth.log file. It's filtering for logs related to the Event ID 4625 or logs related to authentication failures on the auth.log file. Also, it's worth mentioning that since we are doing a cross-platform search, we couldn't narrow down on the `index` search, so we had to use the wildcard * to search across all indexes; 
 - The second line of the query filters out logs that have a null user field on the logs and also, it does something else that we will get into on later charts. To do this the [eval](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/eval) command was used. This is a very capable command that has the ability to transform data and create new fields using existing ones;  
 - The third and fourth line of the query are mere exception rules created to filter out unwanted noise. When the query was being created, I was getting some unwanted noise from poorly parsed logs and these lines are here merely to make sure that the noise stays out. These lines are entirely situational. For this we used the [where](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/where) command which works as a boolean search filter and also the [search](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/search) command to filter out unwanted content from our starting search;  
 - Finally, [timechart](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/timechart) command is here merely to format our data into our timechart.

#### Total Failed Login Attemps Count
Next in line, is the total count of failed login attempts. Even though it essentially presents us with the same information as the previous timechart, this pairs very well with it as it gives us another perspective on the same information, that can be useful in specific situations, that's what dashboards are all about after all.  

<img width="736" height="381" alt="Pasted image 20260707112047" src="https://github.com/user-attachments/assets/20346f86-9e11-4dc7-8d28-bdc66f3928d7" />

As one would expect this is a very simple chart. It's simply a radical of the count we saw in the previous timechart. The query to obtain this will be very similar to the previous one.  
```
index=* (EventCode=4625 OR (source="/var/log/auth.log" "authentication failure"))
| eval User=Case(isnotnull(Account_Name),lower(mvindex(Account_Name,-1)), isnotnull(user), lower(user))
| search User!="win11-client"
| where NOT like(User,"domain/%")
| stats count as "Failed Logons"
```
**Query Analysis:**  
 - The only line that differs from the last query is the last line. Here the [stats](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/stats) command was used, together with the argument `count` to simply count the number of events that match our search query, resulting on the number obtained. The `stats` command is a very powerful statistics command that can be used in many different ways.

#### Failed Login attempts by User and Host

