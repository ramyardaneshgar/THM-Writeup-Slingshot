# THM-Writeup-Slingshot
Writeup for TryHackMe Slingshot Lab - performing forensic log analysis using the ELK stack to reconstruct an attacker's kill chain, identify exploited vulnerabilities, and extract critical indicators of compromise (IOCs).

By Ramyar Daneshgar 

Certainly! Here’s an enhanced and more detailed version of the walkthrough, including specific tools, commands, and steps I used during the investigation.


---

## **1. Identifying the Attacker's IP Address**

### Tools/Commands:
- **Kibana**: Used for filtering logs and analyzing patterns.

### Steps:
1. I logged into the Kibana dashboard and focused on the `apache_logs` dataset.
2. Set the time range to the attack period: **July 26, 2023**.
3. Filtered the logs using:
   ```
   transaction.remote_address: *
   ```
   This showed all client IPs making requests to the server.
4. Sorted the results by the **highest number of requests**.
   
### Outcome:
- The attacker’s IP was identified as **`10.0.2.15`**, with significantly higher activity than other IPs.

---

## **2. Determining the Initial Scanner Used**

### Tools/Commands:
- **Kibana**: Focused on the `request.headers.User-Agent` field.
- **Filters**: Searched specifically for entries from `10.0.2.15`.

### Steps:
1. Filtered the logs:
   ```
   transaction.remote_address: 10.0.2.15
   ```
2. Sorted entries by timestamp to locate the **earliest activity**.
3. Inspected the `request.headers.User-Agent` field.

### Outcome:
- The attacker used the **Nmap Scripting Engine**, as indicated by the User-Agent string. Nmap is a widely used reconnaissance tool.

---

## **3. Identifying the Directory Enumeration Tool**

### Tools/Commands:
- **Kibana**: Analyzed requests with specific User-Agent patterns.
- **Log Filtering**: Focused on sequential requests to common directory names.

### Steps:
1. Continued filtering logs for `10.0.2.15`.
2. Identified sequential requests targeting paths like `/admin`, `/login`, `/uploads`.
3. Observed the User-Agent string:
   ```
   Mozilla/5.0 (Gobuster)
   ```

### Outcome:
- The attacker used **Gobuster**, a directory brute-forcing tool, to enumerate directories.

---

## **4. Counting Failed Resource Requests**

### Tools/Commands:
- **Kibana Query**: Filtered logs for 404 errors.

### Steps:
1. Applied a filter for the attacker’s IP and `404` status codes:
   ```
   transaction.remote_address: 10.0.2.15 AND response.status: 404
   ```
2. Counted the number of entries returned.

### Outcome:
- A total of **1,867 failed resource requests** were identified, highlighting the extent of their enumeration.

---

## **5. Discovering the Flag in an Interesting Directory**

### Tools/Commands:
- **Kibana Query**: Searched for successful directory access (`200` status codes).
- **Steps**:
  1. Filtered the logs for:
     ```
     transaction.remote_address: 10.0.2.15 AND response.status: 200
     ```
  2. Identified an interesting directory accessed by the attacker.
  3. Retrieved a flag: `a76637b62ea99acda12f5859313f539a`.

---

## **6. Discovering the Admin Login Page**

### Tools/Commands:
- **Kibana Logs**: Continued analyzing the attacker's `200` status requests.

### Steps:
1. Traced back successful requests to identify a hidden login page:
   ```
   /admin-login.php
   ```
2. This endpoint was targeted in later phases of the attack.

---

## **7. Identifying the Brute-Force Tool**

### Tools/Commands:
- **Hydra**: A brute-force tool used by the attacker, evident in the logs.
- **User-Agent Analysis**: 
   ```
   Mozilla/4.0 (Hydra)
   ```

### Steps:
1. Filtered logs for `POST` requests to `/admin-login.php` from `10.0.2.15`.
2. Observed repeated attempts with different credentials.

### Outcome:
- The attacker brute-forced the credentials `admin:thx1138` using Hydra.

---

## **8. Retrieving the Uploaded Web Shell**

### Tools/Commands:
- **Kibana Query**: Tracked file uploads by analyzing `POST` requests to `/upload.php`.

### Steps:
1. Filtered logs for requests made to `/admin/upload.php`.
2. Inspected the `request.body` field to uncover the uploaded file's contents.
3. Extracted a flag embedded in the shell:
   ```
   THM{ecb012e53a58818cbd17a924769ec447}
   ```

---

## **9. Identifying the First Command Executed**

### Tools/Commands:
- **Kibana Query**: Analyzed logs from the web shell.

### Steps:
1. Filtered logs for access to `easy-simple-php-webshell.php`.
2. Identified the first command executed:
   ```
   whoami
   ```

### Outcome:
- The attacker confirmed their access and privileges on the system.

---

## **10. Locating the Extracted Database Credentials**

### Tools/Commands:
- **LFI Exploitation**: The attacker exploited a Local File Inclusion (LFI) vulnerability.

### Steps:
1. Identified attempts to access `/etc/phpmyadmin/config-db.php`.
2. Retrieved database credentials:
   ```
   db_user: root
   db_pass: 1234
   ```

---

## **11. Accessing the Database Manager**

### Tools/Commands:
- **phpMyAdmin**: The attacker used the stolen credentials to log into `/phpmyadmin`.

### Steps:
1. Filtered logs for requests to `/phpmyadmin`.
2. Confirmed successful login with the extracted credentials.

---

## **12. Discovering the Exported Database**

### Tools/Commands:
- **Database Export Logs**: Monitored database export activity.

### Steps:
1. Filtered logs for requests to export functionality.
2. Identified a database named `sensitive_data`.

---

## **13. Discovering the Inserted Flag**

### Tools/Commands:
- **SQL Injection Logs**: Analyzed database insert queries.

### Steps:
1. Filtered logs for `INSERT` queries.
2. Found a flag inserted into the database:
   ```
   THM{d41d8cd98f00b204e9800998ecf8427e}
   ```

---

## **Lessons Learned**

### 1. **Effective Log Analysis**
   - Tools like Kibana are essential for tracing attacks in real-time and post-event analysis.

### 2. **Reconnaissance Detection**
   - Monitoring unusual User-Agent strings or high request volumes can help detect attacks early.

### 3. **Mitigating Brute Force**
   - Use rate limiting, account lockout policies, and multi-factor authentication to protect login pages.

### 4. **Preventing LFI Attacks**
   - Sanitize user inputs and restrict access to sensitive files to mitigate LFI vulnerabilities.

