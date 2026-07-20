# Authentication Dashboard

This dashboard is made of three different tabs: 
  - The "general" tab which will give an overview over the environment;
  - The "windows" tab which will focus on the windows machines;
  - The "linux" tab which will focus on the linux machines.

Observations: 
 - The time filter for this dashboard is set to match the attack ran on the Simulation 1 page;
 - Also, due to the limited available time, the usernames present in the screenshots are not redacted. I'm aware that its a bad practice but considering that the lab is not longer live, its rather irrelevant. It's perfectly fine for demonstration purposes.

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
 - The third and fourth line of the query are mere exception rules created to filter out unwanted noise. When the query was being created, I was getting some unwanted noise from poorly parsed logs and these lines are here merely to make sure that the noise stays out. These lines are entirely situational. For this we used the [where](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/where) command which works as a boolean search filter and also the [search](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/search) command to filter out unwanted content from our starting search. In this specific case, I wanted to filter out logs that were related to a "win11-client" user which, considering that I don't have any user named like that and also that this is a hostname, it was safe to assure that it was a poorly parsed log. Then, in some instances I was getting duplicate logs from usernames with the format "domain/username" and thats where the `where` command came in to filter this unnecessary noise out;  
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
The purpose of this chart is to present the failed login attempts in an organized manner, sorting by User, Host and Failed Login count. Its goal is to help visualize which hosts and users were more affected and have a higher probability of being subjects of something like a brute-force attack.

<img width="738" height="172" alt="Pasted image 20260707112440" src="https://github.com/user-attachments/assets/074fed6e-d706-47a2-b89a-47ad132ecde3" />

According to the table above, it's possible to see that all the failed login attempts happened on linux-based hosts, with 2 users having 8 failed login attempts each, each on its own host. There was also a failed login attempt for the root user, on the debian machine.  
```
index=* (EventCode=4625 OR (source="/var/log/auth.log" "authentication failure"))
| eval User=case(isnotnull(Account_Name), lower(mvindex(Account_Name,-1)), isnotnull(user), lower(user)
| search User!="win11-client"
| where NOT like(User,"domain/%")
| stats count as "Failed Logons" values(host) as Hosts by User
| table User Hosts "Failed Logons"
| sort - Failed Logons"
```
**Query Analysis:**
 - The first four lines are the same as our previous queries, but this time around, it's important to explain the full function of the second line of our query. Besides using the `eval` command with the `isnotnull` argument to filter out logs with null User field (`Account_Name` for Windows logs and `user` for Linux logs), it also uses the `lower` argument to format all the usernames into lowercase. Something that was happening specifically on the Windows logs was that the `Account_Name` field was being populated twice, first by the domain name and then with the username. To correct this, the `mvindex` argument was used with the parameter `Account_Name,-1` to only parse last field present in `Account_Name` (username) and ignore the first field (domain name);
 - Once again we make use of the `stats count` command, not only to label our count as "Failed Logons", but also to transform the data. Here we are essentially grouping the data by User, listing all unique hosts where those failures occured and counting said failures. Simply put, counting the failed attempts and labeling them as "Failed Logons" (`stats count as "Failed Logons"`), correlating with "host" value, labeled as "Hosts" (`values(host) as Hosts`), sorted by User (`by User`);
 - Next we use the [table](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/table) command to build the table. This is a very simple, yet effective and poweful command to use. It consists on the command `table`, followed by the columns we wish to have. According to our query, we know we have the "User" field, that we obtain from the `eval` command, and then the "Hosts" and "Failed Logons" fields, that we obtain from the `stats` command. Considering this, we can simply use the `table` command, followed by these fields to make a table using these fields as columns;
 - Lastly, the `sort` command is used with the `-` argument to sort the "Failed Logons" column in descending order. Showing in the first line the User and Host with the biggest amount of failed login attempts, to quickly detect any possible problems.

#### Failed SSH Authentication
The following table, is pretty much equal to the previous one, but dedicated to SSH authentication. Since this protocol is very frequently used, both for legitimate and ilegitimate purposes, it makes all the sense in the world to give it some attention.

<img width="739" height="176" alt="Pasted image 20260707112648" src="https://github.com/user-attachments/assets/53d248b7-5e0f-4108-a221-b1fa91ba3db0" />

The photo above shows us the same information as the one from the previous table. It makes complete sense, since on the attack simulation, there were only failed authentication attempts via SSH on the linux environments.
```
index=* source="/var/log/auth.log" "Failed Password"
| rex field=_raw "for (invalid user)?(?<User>\S+)"
| stats count as Attempts values(host) as Host by User
| table User Host Attempts
| sort -Attempts
```
**Query Analysis:**
 - Previously, while studying the different types of logs, I realized that the pertinent logs for this were the logs in the `auth.log` file that had the expression "Failed Password", hence why the initial filter on the first line of the query;
 - On the next line we make use of the [rex](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/rex) command. This is utilized to extract fields making use of regular expressions. The `field=_raw` argument, tells splunk to search the entire text in the log. Then we start with the parsing, knowing that the log can either be "Failed password for username" or "Failed password for invalid user username" we have to create an expression capable of parsing out the username out of both of these expressions. We start by matching the word "for" on the expression and then add `(invalid user)?`, here the `()?` mean that this part of the sentence is entirely optional, but it's important to let Splunk know that it can be there. Then, the important part, we capture our username with `(?<User>\S+)` and store it in a variable named "User". The `\S+` is added to make Splunk read characters until it reaches a whitespace, thats when the username ends;
 - The next lines of the query follow the exact same logic of the previous one.

#### User Creation
The following table is meant to monitor user creation. This is a popular method of obtaining persistence by attackers and usually only possible to do by using privileged users. Maintaining an eye on this, can help detect signs of persistence and also privileged account compromise.

<img width="737" height="335" alt="Pasted image 20260707112909" src="https://github.com/user-attachments/assets/e1b29657-d1e4-449e-b9dd-58a967935b45" />

```
index=* (Eventcode=4720 OR (source="/var/log/auth.log" "new user:"))
| rex field=_raw "name=(?<LinuxUser>[^,]+)"
| eval WindowsUser=if(EventCode=4720,mvindex(Account_Name,-1),null())
| eval CreatedUser=coalesce(WindowsUser, LinuxUser)
| eval Platform=if(EventCode=4720,"Windows","Linux")
| where isnotnull(CreatedUser)
| table _time Platform host CreatedUser
| sort -_time
```
**Query Analysis:**
 - To monitor for user creation, we have to either take a look at [Windows Event ID 4720](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4720) logs or, for linux, the `auth.log` file, in the logs containing "new user:". This is what the first line of the query filters for;
 - Once, again we use the `rex` command to parse the raw log. Here, we tell Splunk to parse the username after "new user:" and store inside the "LinuxUser" variable. With `[^,]+` we tell it to keep reading characters on the raw text until it reaches a comma, that's when we know the username ends;
 - Then, making use of `eval`, we parse the new windows user and store it inside the "WindowsUser" variable. To do this we use an `if` statement, by filtering on the 4720 ID logs, for the second value stored on the `Account_Name` variable. We filter for the second value because we were facing the same problem as before, where the first value was not the username but the domain name;
 - Next, we create a new variable, "CreatedUser", and make use of `coalesce` to join the previously created variables "WindowsUser" and "LinuxUser", under this new variable;
 - Then, using `eval` once again, we create a new field, "Platform", and use an `if` statement to give it a value. If we are dealign with an `EventCode=4720`, that means its a Windows Event ID 4720 log, meaning we are dealing with a Windows machine. If not, we are dealing with a Linux;
 - Like in previous queries, we make use of the `where` command, together with "isnotnull" expression, to filter out null values, this time on the "CreatedUser" field;
 - Finally, for the last 2 query lines we make use of the `table` command to create a table with the created variables, together with "time" and "host". In the last line we use the command `sort` to show the most recent events first.
