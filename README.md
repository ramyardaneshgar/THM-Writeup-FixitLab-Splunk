# THM-Writeup-FixitLab-Splunk
Writeup for TryHackMe Fixit Lab - Splunk log parsing, event boundary fixes, custom field extraction with regex, and event analysis 

By Ramyar Daneshgar 


## Introduction

The FixIt room on TryHackMe focuses on log ingestion, event boundary definition, field extraction, and event analysis using Splunk. This writeup details my approach to resolving log parsing issues, configuring Splunk for efficient field extraction, and analyzing parsed data for forensic investigation.

---

## Level 1: Fix Event Boundaries

### Task

The log ingestion process in Splunk was failing to determine event boundaries due to multi-line log entries coming from an unknown source. My first task was to fix this issue and define clear boundaries for individual log events.

### Analysis

I analyzed the log entries and observed that they all started with a consistent pattern: `[Network-log]:`. This pattern was unique and could serve as a delimiter to mark the beginning of each log entry. Without properly defining these boundaries, Splunk was treating multi-line logs as single, incomplete events, which rendered them unsearchable and unstructured.

### Steps Taken

1. **Configuration Changes**:
   - I edited the `props.conf` file to define event boundaries using the `BREAK_ONLY_BEFORE` directive. This setting allowed Splunk to treat any line starting with `[Network-log]:` as the beginning of a new event.
   - The configuration in `props.conf`:
     ```conf
     [network_logs]
     SHOULD_LINEMERGE = true
     BREAK_ONLY_BEFORE = \[Network-log\]
     ```

2. **Testing and Restart**:
   - To apply the changes, I restarted Splunk using the following command:
     ```bash
     /opt/Splunk/bin/splunk restart
     ```

3. **Validation**:
   - After restarting, I verified that the multi-line logs were correctly parsed into individual events by searching for `source="network_logs"` in the Splunk console.

### Outcome

Splunk successfully recognized and separated each log entry based on the `[Network-log]:` delimiter. This ensured that each event was complete and properly ingested for further processing.

---

## Level 2: Extract Custom Fields

### Task

Once the events were correctly parsed, the next step was to extract critical fields such as:
- `Username`
- `Country`
- `Source_IP`
- `Department`
- `Domain`

These fields would make the logs searchable and allow for structured analysis.

### Analysis

The log entries followed a predictable structure, as shown in the example below:
```plaintext
[Network-log]: User named Johny Bil from Development department accessed the resource Cybertees.THM/about.html from the source IP 192.168.0.1 and country Japan at: Thu Sep 28 00:13:46 2023
```

Key field observations:
- Each log entry contained the user’s **name**, **department**, **domain** accessed, **source IP address**, and **country**.
- These details could be extracted using regex patterns tailored to match specific parts of the log format.

### Steps Taken

1. **Regex Pattern Development**:
   - Using **Regex101**, I crafted a regex pattern to capture the required fields:
     ```regex
     \[Network-log\]:\sUser\snamed\s([\w\s]+)\sfrom\s([\w]+)\sdepartment\saccessed\sthe\sresource\s([\w]+\.[\w]+\/[\w-]+\.[\w]+)\sfrom\sthe\ssource\sIP\s((?:[0-9]{1,3}\.){3}[0-9]{1,3})\sand\scountry\s([\w\s]+)
     ```
   - This pattern captures:
     - `Username`: Group 1
     - `Department`: Group 2
     - `Domain`: Group 3
     - `Source_IP`: Group 4
     - `Country`: Group 5

2. **Configuring Field Extraction**:
   - Edited `transforms.conf` to define field extraction rules:
     ```conf
     [network_custom_fields]
     REGEX = \[Network-log\]:\sUser\snamed\s([\w\s]+)\sfrom\s([\w]+)\sdepartment\saccessed\sthe\sresource\s([\w]+\.[\w]+\/[\w-]+\.[\w]+)\sfrom\sthe\ssource\sIP\s((?:[0-9]{1,3}\.){3}[0-9]{1,3})\sand\scountry\s([\w\s]+)
     FORMAT = Username::$1 Department::$2 Domain::$3 Source_IP::$4 Country::$5
     ```
   - Configured `fields.conf` to index the extracted fields for efficient querying:
     ```conf
     [Username]
     INDEXED = true

     [Country]
     INDEXED = true

     [Source_IP]
     INDEXED = true

     [Department]
     INDEXED = true

     [Domain]
     INDEXED = true
     ```

3. **Restarting Splunk**:
   - Applied the changes by restarting Splunk:
     ```bash
     /opt/Splunk/bin/splunk restart
     ```

4. **Validation**:
   - I navigated to the Splunk search console and queried the newly extracted fields:
     ```spl
     index=fixit | stats count by Username, Country, Department, Domain, Source_IP
     ```

### Outcome

The custom fields were successfully extracted, making the logs structured and searchable by attributes like `Username`, `Country`, `Domain`, `Department`, and `Source_IP`.

---

## Level 3: Event Analysis

### Task

Using the extracted fields, I analyzed the logs to answer several questions about user activity, resource access, and geographic distribution.

### Analysis Process

1. **Captured Domain**:
   - Query to find all captured domains:
     ```spl
     index=fixit | stats count by Domain
     ```
   - Result: `cybertees.thm`

2. **Top Two Countries for User Robert**:
   - Query:
     ```spl
     index=fixit Username="Robert" | stats count by Country | sort -count
     ```
   - Top countries: `Canada, United States`

3. **Users Accessing Specific Resources**:
   - Query to identify the user accessing `secret-document.pdf`:
     ```spl
     index=fixit Domain="secret-document.pdf" | stats values(Username)
     ```
   - Result: `Sarah Hall`

4. **Summary Metrics**:
   - Number of countries:
     ```spl
     index=fixit | stats dc(Country)
     ```
     Result: `12`
   - Number of departments:
     ```spl
     index=fixit | stats dc(Department)
     ```
     Result: `6`
   - Number of usernames:
     ```spl
     index=fixit | stats dc(Username)
     ```
     Result: `28`
   - Number of source IPs:
     ```spl
     index=fixit | stats dc(Source_IP)
     ```
     Result: `52`

5. **Configuration Files Used**:
   - Files in alphabetical order:
     ```plaintext
     fields.conf, props.conf, transforms.conf
     ```

---

## Lessons Learned

1. **Effective Use of Splunk Configuration Files**:
   - Understanding the purpose and interaction of `props.conf`, `transforms.conf`, and `fields.conf` is essential for log ingestion and structured analysis.

2. **Regex for Log Parsing**:
   - Creating precise regex patterns is critical for extracting meaningful data from semi-structured logs.

3. **Event Boundary Definition**:
   - Properly defining event boundaries ensures multi-line logs are parsed accurately and can be queried efficiently.

4. **Scalability in Log Analysis**:
   - Structuring logs with indexed fields simplifies complex queries, making it easier to identify trends and anomalies.

By fixing event boundaries, extracting structured fields, and conducting detailed analysis, I successfully completed the FixIt room. This lab taught me the importance of proper log management and Splunk configurations in cybersecurity.
