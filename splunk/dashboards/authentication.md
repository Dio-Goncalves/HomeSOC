# Authentication Dashboard

This dashboard is made of three different tabs: 
  - The [General tab](#General-tab) which will give an overview over the environment;
  - The [Windows tab](#Windows-tab) which will focus on the windows machines;
  - The [Linux tab](#Linux-tab) which will focus on the linux machines.

# Contents
1. [General tab](#General-tab)
   - [Overview](#Overview)
   - [In-Depth Analysis](#In-Depth-Analysis)
     - [Total Failed Login Attempts Timechart](#Total-Failed-Login-Attempts-Timechart)
     - [Total Failed Login Attempts Count](#Total-Failed-Login-Attempts-Count)
     - [Failed Login attempts by User and Host](#Failed-Login-attempts-by-User-and-Host)
     - [Failed SSH Authentication](#Failed-SSH-Authentication)
     - [User Creation](#User-Creation)
     - [Deleted Users](#Deleted-Users)
     - [Password Changes](#Password-Changes)
2. [Windows tab](#Windows-tab)
   - [Windows Overview](#Windows-Overview)
   - [Windows In-Depth Analysis](#Windows-In-Depth-Analysis)
     - [Failed Logons Over Time Timechart](#Failed-Logons-Over-Time-Timechart)
     - [Total Failed Logons Count](#Total-Failed-Logons-Count)
     - [Total Failed Logons by User and Host](#Total-Failed-Logons-by-User-and-Host)
     - [User Creation](#User-Creation)
     - [Deleted Users](#Deleted-Users)
     - [Password Changes](#Password-Changes)
     - [Kerberoasting Detection](#Kerberoasting-Detection)
     - [Group Membership Changes](#Group-Membership-Changes)
     - [Kerberos / Ticket Activity](#Kerberos-/-Ticket-Activity)
     - [Failed Kerberos](#Failed-Kerberos)
     - [TGT requests](#TGT-requests)
     - [Mimikatz Execution](#Mimikatz-Execution)
3. [Linux tab](#Linux-tab)
   - [Linux Overview](#Linux-Overview)
   - [Linux In-Depth Analysis](#Linux-In-Depth-Analysis)
     - [Failed Logons Over Time](#Failed-Logons-Over-Time)
     - [Total Failed Logons](#Total-Failed-Logons)
     - [Failed Logons by User and Host](#Failed-Logons-by-User-and-Host)
     - [Failed SSH Logons](#Failed-SSH-Logons)
     - [Locked and Unlocked User Accounts](#Locked-and-Unlocked-User-Accounts)
     - [User Creation](#User-Creation)
     - [Deleted Users](#Deleted-Users)
     - [Password Changes](#Password-Changes)
     - [Sudo Activity](#Sudo-Activity)
     - [Sudo Administrative Activity](#Sudo-Administrative-Activity)

**Observations:** 
 - The time filter for this dashboard is set to match the attack ran on the Simulation 1 page;
 - Also, due to the limited available time, the usernames present in the screenshots are not redacted. I'm aware that its a bad practice but considering that the lab is not longer live, its rather irrelevant. It's perfectly fine for demonstration purposes;
 - I'll only explain something in detail the first time I mention it. A lot of commands will be repeated for different queries, so it doesn't make sense to waste time writing the same thing all the time. If you still have any doubts after reading my explanations, make sure to `ctrl + F` for the command you are looking for or simply make use of the documentation I mention below;
 - Make sure to use the following documentation for reference: [Windows Security Log Events](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/). [Splunk search commands](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/abstract).

## General tab
### Overview
<img width="1431" height="870" alt="image" src="https://github.com/user-attachments/assets/de93cfed-e370-4947-9618-c123ad1d3aef" />

[Back to top](#Contents)

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

[Back to top](#Contents)

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

[Back to top](#Contents)

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

[Back to top](#Contents)

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

[Back to top](#Contents)

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

[Back to top](#Contents)

#### Deleted Users
The following table will monitor for deleted users across all the machines. This is very important to monitor with user tampering and avoid any data loss or control over the machines.

<img width="734" height="312" alt="Pasted image 20260707113145" src="https://github.com/user-attachments/assets/81cb2a93-2752-4039-a70d-72dbf7707ec4" />

```
index=* EventCode=4726
| eval Platform="Windows"
| eval DeletedUser=mvindex(Account_Name,-1)
| table _time Platform host DeletedUser
| append [
search index=* source="/var/log/auth.log" ("userdel" OR "deluser")
| rex field=_raw "delete user '(?<DeletedUser>[^']+)'"
| eval Platform="Linux"
| where isnotnull(DeletedUser)
| table _time Platform host DeletedUser
]
| sort -_time
```
**Query Analysis:**
 - The approach is similar to the previous query but here we will make use of the [append](https://help.splunk.com/en/splunk-enterprise/spl-search-reference/9.2/search-commands/append) command to have a more organized query. As the name suggests, this allows us to append one search to another, allowing us to effectively stack two searches on top of each other, while querying for them in a more organized way;
 - Starting with the first search, we search for the [Windows Event ID 4726](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4726) logs, related to deleting user accounts;
 - The logic on the following lines of the query is very similar to the one previously seen, we'll tell Splunk to create a "Platform" variable and store the "Windows" value inside it every time it sees the `EventCode=4726` value. Then parse out the second value stored inside `Account_Name` field, to obtain our username, store it inside the "DeletedUser" variable and create a table with the created variables, together with "time" and "host" variables;
 - Then, we make use of the `append` command to essentially attach a similar search to the one we just did, but for Linux machines. We search the `auth.log` file for "userdel" or "deluser", as this can differ between linux and debian machines;
 - Then use the `rex` command to parse out the username from the raw log file, utilizing similar logic as before and storing the value in the "DeletedUser" variable;
 - Once again, using the `eval` command, we tell Splunk to store the "Linux" value inside the "Platform" variable;
 - We filter out null values with the `where` command and then proceed to create a table similar to the one in the first part of the search with the same columns. It's important to have matching columns to have a consistent table;
 - Using the `sort` command, we sort the events to show the most recent ones first.

[Back to top](#Contents)

#### Password Changes
The last table on this tab will monitor password changes and password resets. This is another popular method for attackers lock the victims out of their own machines.

<img width="1490" height="246" alt="Pasted image 20260707113540" src="https://github.com/user-attachments/assets/6bb97740-8885-4dd7-bbcb-ba23f535950a" />

```
index=* (EventCode=4723 OR EventCode=4724)
| eval Platform="Windows"
| eval User=mvindex(Account_Name,-1)
| eval Action=case(EventCode=4723,"Password Changed", EventCode=4724,"Password Reset")
| where isnotnull(User)
| table _time Platform host User Action
| append [
search index=* source="/var/log/auth.log" "password changed for"
| rex field=_raw "password changed for (?<User>\S+)"
| eval Platform="Linux"
| eval Action="Password Changed"
| table _time Platform host User Action
]
| sort -_time
```
**Query Analysis:**
 - This query makes use of logic similar to what was used so far, using functions like `eval` to create new variables from existing data, `rex` to parse raw logs and `append` to split the search;
 - Starting with the first part of the search, related to Windows, we start by searching for events with [Windows Event ID 4723](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4723), related to password changes, and [Windows Event ID 4724](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4724), related to password resets;
 - Then, using the command `eval`, we tell Splunk that if it finds these events, to create a variable named "Platform" and store the value "Windows" in it. Once again, also using `eval`, we will create a variable named "User" and store there the last value from the "Account_Name" field on the logs (remember from previous queries that this is because I had 2 values stored on this field, the first value was the domain name and the second value was the username);
 - Also using the `eval` command, we create an "Action" variable and by using the `case` command, we'll tell Splunk to store different values in it, depending on the type of Windows Event ID we find on our log. This will serve as a column on the table we'll build for a nice quick visual reference to understand which action was performed on the target account;
 - Then, using the `where` function we filter the null values for the "User" variable and finally, using the `table` command, we create a table with the variables we just created using `eval`, together with the existing "time" and "host" fields;
 - Then, using the `append` command, we'll attach to our query the second part of it, related to Linux. We start by searching for the `auth.log` file and the expression "password changed for" to filter for the right logs;
 - Like in previous queries, we make use of the `rex` command to parse the raw logs and store the username on the "User" variable";
 - Equal to the first part of the search, we use the `eval` command to create the "Platform" variable, assigning it the value of "Linux" and the "Action" variable, with the "Password Changed" value. This means that if we are dealing with Linux logs, these variables will take these values automatically;
 - Finally, using the `table` command, we create a table with the same columns as the one we created on the Windows search and sort the events to show the newest ones first, by using the command `sort`.

[Back to top](#Contents)

## Windows tab  
### Overview
<img width="1885" height="597" alt="Pasted image 20260707111135" src="https://github.com/user-attachments/assets/9fe421cc-dcd2-4172-9023-21177efdb5bb" />
<img width="1867" height="537" alt="Pasted image 20260707111148" src="https://github.com/user-attachments/assets/734543d7-c0e4-4006-b47c-7ccaa48537e2" />
<img width="1863" height="427" alt="Pasted image 20260707111205" src="https://github.com/user-attachments/assets/50d832dc-ef31-4135-a061-3d01b5d08712" />
<img width="1860" height="388" alt="Pasted image 20260707111218" src="https://github.com/user-attachments/assets/523ed96a-d58a-469f-8824-8d1dcbccfbaa" />
<img width="1861" height="388" alt="Pasted image 20260707111230" src="https://github.com/user-attachments/assets/e71640a7-e15f-4449-aa1c-b5a64e761504" />
<img width="1856" height="381" alt="Pasted image 20260707111239" src="https://github.com/user-attachments/assets/11b17bb9-d40c-43fb-9509-265d4d105212" />

[Back to top](#Contents)

### In-Depth Analysis
#### Failed Logons Over Time Timechart
Once again, the dashboard starts with a timechart of total failed login attempts. This time, its specific to Windows.  

<img width="920" height="381" alt="Pasted image 20260708083529" src="https://github.com/user-attachments/assets/e272de01-be5a-4c3c-85ba-67319aa86080" />

Similarly to the first query on the previous tab, here we once again made use of the [Windows Event ID 4625](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4625) (An account failed to log on). Since we are only dealing with Windows events, this time the SPL query will be even simpler:  
```
index=* EventCode=4625 Failure_Reason=*Unknown user name or bad password.*
| timechart count
```
**Query Analysis:**
 - Analyzing the query, besides the obvious `EventCode=4625` filter, we also filter for the expression `Failure_Reason=*Unknown user name or bad password.*` to specifically filter for bad credential events and filter out any additional noise that may come out;
 - Once again we use the `timechart count` command to format the data into a timechart with the event count.

[Back to top](#Contents)

#### Total Failed Logons Count
Similarly to the previous tab of the dashboard, next in line is the total count of failed login attempts presented in the form of a radical, allowing us to have another perspective on the information.

<img width="918" height="181" alt="Pasted image 20260708083541" src="https://github.com/user-attachments/assets/27681782-a4a3-4ce8-a255-adbf7bd17231" />

The SPL query is very similar to the one that presents us the timechart, the only difference being in the piped command. For this query we'll make use of the `stats count` command to obtain a radical number with the total amount of events:
```
index=* EventCode=4625 Failure_Reason=*Unknown user name or bad password.*
| stats count as "Total Failed Logons"
```
**Query Analysis:**
 - As mentioned above, the only difference to the previous query is the piped command, where we pipe the `stats count as "Total Failed Logons"` command to obtain the total amount of events with the "Total Failed Logons" label.

[Back to top](#Contents)

#### Failed Logons by User and Host
The next table nicely displays the amount of failed login attempts, separating by user and host. This allows us to evaluate specifically which user and machine is more susceptible to being attacked.

<img width="916" height="183" alt="Pasted image 20260708083553" src="https://github.com/user-attachments/assets/94414490-bdeb-40ec-8919-534b902de343" />

Once again, we made use of the [Windows Event ID 4625](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4625) for the SPL query:
```
index=* EventCode=4625
| eval User=lower(mvindex(Account_Name,-1))
| search NOT User IN ("win11-client","domain/endpoint01")
| stats count as "Failed Logons" values(host) as Host by User
| table User Host "Failed Logons"
| sort - "Failed Logons"
```
**Query Analysis:**
 - This query follows exactly the same logic of the one found in [Failed Login attempts by User and Host](#Failed-Login-attempts-by-User-and-Host). In order to avoid repeating myself, please refer to this one.

[Back to top](#Contents)

#### User Creation
The next table displays the events regarding user creation by host and user, sorted by newer events first. In this specific case, it is possible to see what seem to be duplicate events but they are different events as seen in the timestamps. I merely proceeded to create and delete users to generate telemetry and test everything.

<img width="924" height="233" alt="Pasted image 20260708083630" src="https://github.com/user-attachments/assets/a9626359-0290-4f71-84f2-c33ffdec6196" />

To build the SPL query, we made use of the [Windows Event ID 4720](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4720) (An account was created):
```
index=* EventCode=4720
| eval "New User"=if(EventCode=4720,mvindex(Account_Name,-1),null())
| where isnotnull("New User")
| table _time host "New User"
| sort -_time
```
**Query Analysis:**
 - The first line of the query consists of a simple filter for the logs related to the previously mentioned Windows Event ID 4720;
 - The second line consists of, once again, the usage of the `eval` command to create a new variable "New User". This new variable will be associated with a condition that will search for the last name in the `Account_Name` field in the Event Code 4720 logs;
 - The third line of the query makes use of the `where isnotnull` command to filter out null values for the "New User" variable;
 - Then, we make use of the `table` command to build a table with the time, host and "New User" columns;
 - Finally, in the last line, we use the `sort` command to sort the events by newest first.

[Back to top](#Contents)

#### Deleted Users
The next table displays the events regarding deleted users, in a similar manner to our previous table.

<img width="920" height="235" alt="Pasted image 20260708083639" src="https://github.com/user-attachments/assets/d2423159-936f-46c8-a7b2-e151fa9dca83" />

To build the SPL query, we followed the same exact logic as the previous query. The only difference was the we made use of the [Windows Event ID 4726](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4726) (A user account was deleted):
```
index=* EventCode=4726
| eval "Deleted User"=if(EventCode=4726,mvindex_(Account_Name,-1),null())
| where isnotnull("Deleted User")
| table _time host "Deleted User"
| sort -_time
```
**Query Analysis:**
 - As previously mentioned, this query follows the exact same logic as the previous one. The only difference being on the Windows Event ID used and the new variable name.

[Back to top](#Contents)

#### Password Changes
The following table will monitor changes related to user passwords, being regular password changes or password resets.

<img width="920" height="264" alt="Pasted image 20260708083654" src="https://github.com/user-attachments/assets/c2013079-7bb6-42ed-8285-6569a6136a4a" />

For this query we made use of the [Windows Event ID 4723](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4723) (An attempt was made to change an account's password) and [Windows Event ID 4724](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4724) (An attempt was made to reset an account's password):
```
index=* (EventCode=4723 OR EventCode=4724)
| eval User=mvindex(Account_Name,-1)
| eval Action=case(EventCode=4723,"Password Changed", EventCode=4724,"Password Reset")
| where isnotnull(User)
| table _time host User Action
| sort -_time
```
**Query Analysis:**
 - The first line consists on a very standard filter that filters for both the previously mentioned event IDs;
 - In the second line, using the `eval` command, we create a new variable "User" and assign it the last value of the `Account_Name` field;
 - The third line consists of yet another new variable creation, "Action", which will described if there was a password change or a password reset. To do this, we use the `case` command to build the two possible scenarios. If the `EventCode` field matches 4723, then the "Action" variable will have the "Password Changed" value. If the `EventCode` field matches 4724, then the "Action" variable will have the "Password Reset" value;
 - The following line is a simple filter for null values;
 - Then, we make use of the `table` command to build a table with the time, host, User and Action columns;
 - Finally we sort the events by newest first, using `sort`.

[Back to top](#Contents)

#### Kerberoasting Detection
For this table, I decided to monitor for kerberos service ticket requests and password hash access, to try and detect kerberoasting attempts. _This table in particular should be subject to further review as the result is very noisy, since there are a LOT of service ticket requests happening all the time, making this table borderline useless._

<img width="922" height="278" alt="Pasted image 20260708083703" src="https://github.com/user-attachments/assets/bec7d7e9-7046-4a1d-a2f1-1f486e9032e2" />

For this query we made of the [Windows Event ID 4769](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4769) (A Kerberos Service Ticket was requested) and [Windows Event ID 4782](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4782) (The password hash of an account was accessed):
```
index=* (EventCode=4782 OR EventCode=4769)
| eval User=mvindex(Account_Name,-1)
| eval Action=case(EventCode=4769,"A Kerberos Service ticket was requested", EventCode=4782,"The password hash of an account was accessed")
| table _time host User Action
| sort -_time
```
**Query Analysis:**
 - The first line consists on a simple filter for the previously mentioned Windows Event IDs;
 - Then, a new variable "User" was created, using the `eval` command;
 - Once again using `eval`, a new variable "Action" was created. In a similar fashion to the previous query, we used the `case` command to define a case for each Event ID, changing the value stored in "Action" depending on the Event ID;
 - Then, using the `table` command, a table was created with the time, host, User and Action columns;
 - The last line sorts the events by newest first, using the `sort` command.

[Back to top](#Contents)

#### Group Membership Changes
This table, as the name suggests, monitors group membership changes. This can be an effective way of achieving privilege escalation and obtaining persistence. To do this, we'll monitor:  
 - [Windows Event ID 4728](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4728) - A member was added to a security-enabled global group;
 - [Windows Event ID 4732](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4732) - A member was added to a security-enabled local group;  
 - [Windows Event ID 4756](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4756) - A member was added to a security-enabled universal group.
For more information on the scope of each of these group types, please click [here](https://ss64.com/nt/syntax-groups.html). 

<img width="1850" height="375" alt="Pasted image 20260708083724" src="https://github.com/user-attachments/assets/b78b4870-8e06-456e-859c-e78f14a4a190" />

```
index=* (EventCode=4728 OR EventCode=4732 OR EventCode=4756)
| eval "Group Type"=case(
EventCode=4728,"Global Group",
EventCode=4732,"LocalGroup",
EventCode=4756,"Universal Group"
)
| table _time host Account_Name Group_Name "Group Type"
```
**Query Analysis:**
 - The first line consists on a simple filter for the previously mentioned Windows Event IDs;
 - Using `eval`, a new variable "Group Type" was created. In a similar fashion to the previous query, we also made use of the `case` command to define a case for each Windows Event ID and, depending on the Windows Event ID, we would store a different value in "Group Type";
 - The last line is a simple use of the `table` command to build a table with the time, host, Account_Name, Group_Name and Group Type columns;
 - This query has a clear symptom of the time restraints that this project had, as it could use a filter on the `Group_Name` field to remove unnecessary noise. Maybe filter it to only monitor critical groups like the "Administrators" group.

[Back to top](#Contents)

#### Kerberos / Ticket Activity
This table was meant to monitor general Kerberos ticket activity. I was looking for ways to monitor kerberos activity in general, but looking back it needed further filtering and tuning, since the end result is still too noisy, generating multiple events per minute and giving no clear insight. To build this table we monitored the following:
 - [Windows Event ID 4768](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4768) - A Kerberos authentication ticket (TGT) was requested;
 - [Windows Event ID 4769](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4769) - A Kerberos service ticket was requested;
 - [Windows Event ID 4770](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4770) - A Kerberos service ticket was renewed;
 - [Windows Event ID 4771](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4771) - Kerberos pre-authentication failed;
 - [Windows Event ID 4776](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4776) - The domain controller attempted to validate the credentials for an account.

<img width="919" height="375" alt="Pasted image 20260708083740" src="https://github.com/user-attachments/assets/025b28de-37de-4863-b125-8c5d334ad8b8" />

```
index=* (EventCode=4768 OR EventCode=4768 OR EventCode=4770 OR EventCode=4771 OR EventCode=4776)
| eval User=mvindex(Account_Name,-1)
| eval Activity=case(
EventCode=4768,"TGT Requested",
EventCode=4769,"Service Ticket Requested",
EventCode=4770,"Ticket Renewed",
EventCode=4771,"Pre-Authentication Failed",
EventCode=4776,"NTLM Authentication"
)
| eval Encryption=case(
Ticket_Encryption_Type="0x11","AES128",
Ticket_Encryption_Type="0x12","AES256",
Ticket_Encryption_Type="0x17","RC4-HMAC",
Ticket_Encryption_Type="0x18","RC4-HMAC-EXP",
)
| table _time host User Service_Name Encryption Activity
| sort -_time
```
**Query analysis:**
 - The logic behind building this query is every similar to the previous ones. The first line consists on filtering for the Windows Event IDs we are looking for;
 - Then, the "User" variable was created, making use of the `eval` command. The last value of the `Account_Name` field will be stored in this variable;
 - On the third line, using the `eval` command, the "Activity" variable is created. Here, we'll make use of the `case` command to define a case for each possible Windows Event ID, assigning a value to this variable, depending on the Windows Event ID found;
 - The same logic is applied when creating the "Encryption" variable. This variable will be useful to visually display the encryption type used in the kerberos activity. To filter unnecessary noise, it would be useful to further filter this field to only show events that use RC4 encryption, as its currently deprecated and presents a major risk for kerberoasting;
 - Finally, we built a table with time, host User, Service_Name, Encryption and Activity columns.

[Back to top](#Contents)

#### Failed Kerberos
Even though we've already filtered for the the events in this table, in the previous table, I've decided to create a separate one decicated to Kerberos Authentication Failures. This table will monitor the previously mentioned [Windows Event ID 4771](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4771) and [Windows Event ID 4776](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4776).

<img width="912" height="373" alt="Pasted image 20260708083752" src="https://github.com/user-attachments/assets/300c2cc0-b3a9-426d-ba94-a443405fd2e6" />

```
index=* (EventCode=4771 OR EventCode=4776)
| eval User=mvindex(Account_Name,-1)
| eval Activity=case(
EventCode=4771,"Kerberos Pre-Auth Failure",
EventCode=4776,"NTLM Authentication Failure"
)
| table _time host User Activity
| sort -_time
```
**Query analysis:**
 - The first lines consist on a simple filter for the Windows Event IDs we are looking for;
 - Next, using the `eval` command, we create the "User" variable, populating it with the last value stored on the `Account_Name` field;
 - Once again using the `eval` command, we create the "Activity" variable. Together with the `case` command, we create a case for each possible ID, populating this variable with the appropriate Windows Event ID description;
 - Finally, we create a table with the time, host, User and Activity columns and sort the table by time.

[Back to top](#Contents)

#### TGT Requests
This table monitors TGT requests, by simply monitoring [Windows Event ID 4768](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4768). Even though this event is already monitored in previous tables, I felt like it was important to have a table dedicated to this. Looking back, I would probably change the way to present this information and maybe filter it further in order to make this information more actionable.

<img width="1855" height="371" alt="Pasted image 20260708083815" src="https://github.com/user-attachments/assets/6c0330e3-bfb6-4662-a710-f794db67269f" />

```
index=windows EventCode=4768
| table _time Account_Name Service_Name
```
**Query analysis:**
 - This is a very simple query, consisting on a simple filter for the Windows Event ID we are looking for, followed by the `table` command, which gives us a table with the time, Account_Name and Service_Name columns.

[Back to top](#Contents)

#### Mimikatz Execution
This table will monitor for [mimikatz](https://github.com/gentilkiwi/mimikatz) usage. This is a very well known tool famous for being able to extract plaintext passwords, hashes and kerberos tickets from memory and also performing a plethora of different kerberos attacks.  
To achieve this, I monitored for the `*mimikatz*` expression in the `Image` and `CommandLine` fields of the logs. I've also filtered for the sysmon index.

<img width="1852" height="378" alt="Pasted image 20260708083832" src="https://github.com/user-attachments/assets/4c62d8a7-0e69-4dca-bee7-7e88b22fae51" />

```
index=sysmon (Image="*mimikatz*" OR CommandLine="*mimikatz*")
| eval User=mvindex(User,-1)
| table _time host User Image
```
**Query analysis:**
 - The first line consists on an index filter in order to attain the logs stored in the sysmon index and therefore getting only sysmon logs. Then we also filtered for the `*mimikatz*` expression in the `Image` and `CommandLine` fields of the logs;
 - Then, using the `eval` command, we created the "User" variable, populating it with the last value stored on the `User` field in the log;
 - Finally, we built the table with the time, host User and Image columns, giving us a table that shows when mimikatz is executed, the user that executes it and which machine it is executed in.

[Back to top](#Contents)

## Linux tab
### Overview
<img width="1867" height="425" alt="Pasted image 20260707111541" src="https://github.com/user-attachments/assets/70e75479-415d-4b61-91ad-646d9241e3c6" />
<img width="1866" height="484" alt="Pasted image 20260707111557" src="https://github.com/user-attachments/assets/5c9347db-e133-4497-b112-8a891bb7d923" />
<img width="1865" height="488" alt="Pasted image 20260707111614" src="https://github.com/user-attachments/assets/b3ffceda-740c-46b5-bee3-bf8938bd06ed" />
<img width="1864" height="394" alt="Pasted image 20260707111630" src="https://github.com/user-attachments/assets/fd6e40f9-81e1-4ed4-affb-46c2eee48393" />
<img width="1852" height="401" alt="Pasted image 20260707111648" src="https://github.com/user-attachments/assets/d5b67afa-317c-4c10-9e65-11c94ee99fd4" />

[Back to top](#Contents)

### In-Depth Analysis
#### Failed Logons Over Time
The dashboard begins with a timechart of the failed login attempts over time. This is a nice visual presentation that can quickly indicate if something is off, like a sudden spike in failed attempts.

<img width="734" height="375" alt="Pasted image 20260708084513" src="https://github.com/user-attachments/assets/f4038276-c4cb-434f-af11-37a312fbb551" />

```
index=* source="/var/log/auth.log" "authentication failure"
| where isnotnull("user")
| timechart count
```
**Query analysis:**
 - The first line of the query is a simple filter that queries for the contents of the `auth.log` file. This is a file in Linux that stores all the logs related to authorization and authentication. By analyzing the logs on this file it was also possible to conclude that by adding the expression `"authentication failure"`, we could filter for the failed login attempts, and filter any unwanted noise;
 - The second line of the query simply filters out null values on the `user` field;
 - The last line makes use of the `timechart` command to build a timechart with the count of failed login attempts.

[Back to top](#Contents)

#### Total Failed Logons
Next, we have a counter of the total failed login attempts, in the form of a radical. This presents the same information as the previous timechart, but in a different way, providing a nice visual aid by allowing to quickly detect an unusually high number of failed login attempts.

<img width="734" height="183" alt="Pasted image 20260708084533" src="https://github.com/user-attachments/assets/3ca89def-86d7-4f58-99b5-8ed4155493a1" />

```
index=* source="/var/log/auth.log" "authentication failure"
| stats count as "Total Failed Logons"
```
**Query analysis:**
 - The first line of this query follows exactly the same logic as the previous one;
 - The second line consists on using the `stats count` command to get the total amount of failed login attempts and to label the count as "Total Failed Logons".

[Back to top](#Contents)

#### Failed Logons by User and Host
The following table, monitors for Failed Login Attempts by User and Host. This allows for quick detection of a potential brute force attack and to quickly find the victim username and machine.

<img width="738" height="174" alt="Pasted image 20260708084553" src="https://github.com/user-attachments/assets/3d6466c5-7d2b-49e9-8861-1b8c1d99ac0b" />

```
index=* source="/var/log/auth.log" "authentication failure"
| stats count as "Failed Logons" values(host) as Host by user
| table user Host "Failed Logons"
| sort - "Failed Logons"
```
**Query analysis:**
 - Once again, the first line of this query is exactly the same as the previous queries;
 - The second line makes use of the `stats count` command to count the total amount of failed login attempts, and separate them by Host and User;
 - The third line of the query is a simple usage of the `table` command to build a table with the user, Host and "Failed Logons" columns;
 - Finally, we sort the "Failed Logons" column in descending order, so that the user and machine with the most incidences comes first on the table.

[Back to top](#Contents)

#### Failed SSH Logons
This table, follows the same logic as the previous table but only monitors failed login attempts via SSH. This is a very common vector for brute force attacks so its a good idea to remain vigilant on this port.

<img width="1494" height="185" alt="Pasted image 20260708084625" src="https://github.com/user-attachments/assets/5d82b1d2-3e0e-4e4c-82e4-3f922b17621f" />

```
index=* source="/var/log/auth.log" "Failed Password"
| rex field=_raw "for (invalid user)?(?<User>\S+)"
| stats count as Attempts values(host) as Host by User
| table User Host Attempts
| sort - Attempts
```
**Query analysis:**
 - Once again, analyzing the logs in `auth.log`, I realized that to filter for the failed login SSH logs, I had to filter for the expression `"Failed Password"`. With this in mind, the first line consists on this filter;
 - As these logs were in raw format, the second line makes use of the `rex` command to parse the username out of the raw log and store it in the variable "User";
 - Then, in a similar fashion to the previous query, the third line makes use of the `stats count` command to count the failed login attempts by Host and User;
 - The fourth line makes use of the `table` command to build a simple table with the User, Host and Attempts columns;
 - The last line sorts the Attempts column in descending order, so that we can see the User and Host with the highest amount of failed login attempts first.

[Back to top](#Contents)

#### Locked and Unlocked User Accounts
The following table monitors for Locked and Unlocked User Accounts. This table could be useful in the event of having the users setup in a way so that when they had a certain amount of failed login attempts, they would be locked out. This table could help detect victims of brute force attacks or simply locked users with malicious intent by an attacker in a compromised user.

<img width="1495" height="217" alt="Pasted image 20260708085045" src="https://github.com/user-attachments/assets/4d89b9d0-4621-4dd2-a98a-8fab053439fd" />

```
index=linux sourcetype=auth-2 COMMAND="/usr/sbin/usermod"
| rex field=_raw "COMMAND=/usr/sbin/usermod\s+(?<Action>-[LU]|--lock|--unlock)\s+(?<TargetUser>\S+)"
| eval Activity=case(
Action="-L","Account Locked",
Action="--lock","Account Locked",
Action="-U","Account Unlocked",
Action="--unlock","Account Unlocked"
)
| where isnotnull(TargetUser)
| table _time host User PWD TargetUser Activity COMMAND
| sort -_time
```
**Query analysis:**
 - The first line of the query consists on filtering for the appropriate logs. After analyzing the logs, the best approach was to filter for the linux index, the usermod command with `COMMAND=/usr/sbin/usermod` and auth-2 sourcetype;
 - Since the logs are in raw format, the `rex` command had to be used again. With this command, we filtered for the before mentioned command. We also used a named capture group with `(?<Action>-[LU]|--lock|--unlock)`, which allows us to designate the different possibilities for the executed action (lock or unlock) and store the value in the `Action` variable. After filtering and parsing the `Action` value, we simple parsed the value for the target username and store it under the `TargetUser` variable;
 - Then by using the `eval` command, we create a new variable, "Activity". Then, using the `case` command, we create different scenarios for this variable. Depending on the value of the "Action" variable, a different value will be stored on the "Activity" variable;
 - The following line is a simple filter to exclude null values on the "TargetUser" variable;
 - Then, using the `table` command, a table was built with the time, host User, PWD, TargetUser, Activity and COMMAND columns;
 - Finally the table was sorted by time, to show the most recent events first.

[Back to top](#Contents)

#### User Creation
The following table is meant to simply monitor user creation. This is a very popular method of gaining persistence on the victim machine, so it's important to stay vigilant on this.

<img width="735" height="228" alt="Pasted image 20260708084719" src="https://github.com/user-attachments/assets/dd081148-1ada-4647-b2e0-2856f32e2063" />

```
index=* source="/var/log/auth.log" "new user:"
| rex field=_raw "name=(?<CreatedUser>[^,]+)"
| table _time host CreatedUser
| sort -_time
```
**Query analysis:**
 - The first line filters for the logs in the `auth.log` file with the added expression of `"new user:"`, to filter for the user creation logs. This was once again achieved, by previously analyzing the logs;
 - Since the logs are in raw format, the `rex` command was once again used to parse out the new username out of the log and store it under the variable "CreatedUser";
 - Then, a table was built with the time, host and CreatedUser columns;
 - Finally the table was sorted by time to show the most recent events first.

[Back to top](#Contents)

#### Deleted Users
The next table was meant to monitor deleted users.

<img width="739" height="223" alt="Pasted image 20260708084743" src="https://github.com/user-attachments/assets/d20d79ee-2da9-4c68-8de6-1e35cd2b7c59" />

```
index=* source="/var/log/auth.log" "userdel"
| rex field=_raw "delete user '(?<DeletedUser>[^']+)'"
| search DeletedUser=*
| table _time host DeletedUser
| sort -_time
```
**Query analysis:**
 - Once again, by previously analyzing the logs, the filter of the first line was built. By filtering for the `auth.log` file and `"userdel"` expression, we could obtain the logs related to user deletion;
 - Since the logs are in raw format, we had to parse the Deleted User out of them, by using the `rex` command. The deleted username is stored under the variable "DeletedUser";
 - Next, using the `search` command we filter out any possible noise by only searching for events that populate the "DeletedUser" variable, keeping the null values out;
 - Then we proceed to build the table with the time, host and DeletedUser columns;
 - Finally, we sort the table by time to show the most recent events first.

[Back to top](#Contents)

#### Password Changes
The following table will monitor password changes across users. It's important to keep an eye on events like this since an attacker can gain persistence like this and lock the victim out of their own machine.

<img width="1493" height="163" alt="Pasted image 20260708084806" src="https://github.com/user-attachments/assets/4d32e231-06c0-4d5b-bf66-33da499184e4" />

```
index=* source="/var/log/auth.log" "password changed for"
| rex field=_raw "password changed for (?<TargetUser>\S+)"
| table _time host TargetUser
| sort -_time
```
**Query analysis:**
 - By analyzing the logs, it was possible to find the logs related to password changes by filtering for the `auth.log` file and the expression `"password changed for"`;
 - Since the logs were in raw format, the `rex` command was used to filter out the user which the password was changed and store it under the variable "TargetUser";
 - Then a table was built with the time, host and TargetUser columns;
 - Finally the table was sorted by time to show the most recent events first.

[Back to top](#Contents)

#### Sudo Activity
The following table monitors for sudo activity across the linux machines. Given the power that this command has, its very important to monitor activity related to it, considering that a compromised sudo user can be a very big problem.

<img width="740" height="344" alt="Pasted image 20260708084844" src="https://github.com/user-attachments/assets/c4ce6cde-001f-4533-8a76-12894873da77" />

```
index=* source="/var/log/auth.log" "sudo:"
| table _time host USER PWD COMMAND
| where isnotnull(COMMAND)
| sort -_time
```
**Query analysis:**
 - The first line filters for the logs related with the usage of the sudo command. Once again filtering for the `auth.log` file and the `"sudo:"` expression;
 - Then a table is built with the host, USER, PWD and COMMAND columns;
 - On the third line, we filter out the events with null values on the COMMAND field;
 - Then, the table is sorted by time to show the most recent events first.

[Back to top](#Contents)

#### Sudo Administrative Activity
This table monitors sudo activity, in a similar way to the previous one. The difference is that this one is more descriptive. Looking back, the existence of these two tables is debatable, considering that they showcase the same information in a slightly different way. To reduce visual clutter, it would be wise to keep this table and delete the previous one.

<img width="735" height="375" alt="Pasted image 20260708084859" src="https://github.com/user-attachments/assets/c823bbc4-59cc-457f-adda-bba7df013d7a" />

```
index=* source="/var/log/auth.log" "sudo:"
| eval Activity=case(
like(_raw,"%useradd%"),"User Created",
like(_raw,"%adduser%"),"User Created",
like(_raw,"%userdel%"),"User Deleted",
like(_raw,"%deluser%"),"User Deleted",
like(_raw,"%usermod -L%"),"User Locked",
like(_raw,"%usermod -U%"),"User Unlocked",
like(_raw,"%COMMAND=/usr/bin/passwd%"),"Password Changed",
)
| rex field=_raw "usermod\s+-[LU]\s+(?<TargetUser>\S+)"
| rex field=_raw "useradd\s+(?<TargetUser>\S+)"
| rex field=_raw "adduser\s+(?<TargetUser>\S+)"
| rex field=_raw "deluser\s+(?:--remove-home\s+)?(?<TargetUser>\S+)"
| rex field=_raw "userdel(?:\s+-\S+)*\s+(?<TargetUser>\S+)"
| rex field=_raw "passwd\s+(?<TargetUser>\S+)"
| where isnotnull(Activity)
| table _time host USER TargetUser Activity COMMAND
| sort -_time
```
**Query analysis:**
 - The first line consists on the same filter used in the previous query.
 - On the second line we make use of the `eval` command, together with `case` to create a new variable "Activity" and store a descriptive value in it, depending on the parsed command;
 - Then, since these logs are in raw format, the `rex` command was used to parse out the possible administrative sudo commands and store the target username under the new variable, "TargetUser";
 - Using `where isnotnull(Activity)`, any logs with null values on the "Activity" variable are filtered out;
 - Then, a table was built with the time, host, USER, TargetUser, Activity and COMMAND columns;
 - Finally, the table was sorted by time to show the most recent events first.

[Back to top](#Contents)
