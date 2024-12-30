
# **THM-Writeup-Slingshot**  
**Writeup for TryHackMe Slingshot Lab - performing forensic log analysis using the ELK stack to reconstruct an attacker's kill chain, identify exploited vulnerabilities, and extract critical indicators of compromise (IOCs).**  

**By Ramyar Daneshgar**  

---

## **1. Identifying the Attacker's IP Address**

### Tools/Commands:
- **Kibana**: Filter and query logs using custom filters and KQL (Kibana Query Language).

### Steps:
1. Navigated to the **Kibana Dashboard** and focused on the `apache_logs` dataset, which contained detailed HTTP logs.
2. Set the time filter to cover the attack window: **July 26, 2023**.
3. Used the query:
   ```kql
   transaction.remote_address: *
   ```
   This displayed all requests grouped by their originating IP addresses.
4. Sorted the data by request volume to find the top contributors.

### Insights:
- The IP `10.0.2.15` emerged as the attacker’s IP, generating a significantly higher number of requests than any other source.
- High request volume is often indicative of reconnaissance or automated tools, both of which generate numerous queries.

### Outcome:
- The attacker’s IP was confirmed as **`10.0.2.15`**, a critical starting point for tracing their actions.

---

## **2. Determining the Initial Scanner Used**

### Tools/Commands:
- **Kibana**: Queries and filters for analyzing `User-Agent` strings in the logs.

### Steps:
1. Filtered the logs for the attacker’s IP using:
   ```kql
   transaction.remote_address: 10.0.2.15
   ```
2. Sorted entries by timestamp in ascending order to locate the earliest recorded activities.
3. Focused on the `request.headers.User-Agent` field to identify tools used by the attacker.
4. Observed entries containing the following User-Agent string:
   ```
   Nmap Scripting Engine
   ```

### Insights:
- The attacker used **Nmap’s Scripting Engine**, a well-known reconnaissance tool.
- Nmap enables attackers to map open ports, services, and potentially exploitable configurations.
- Early usage of Nmap indicates an attempt to profile the server before executing any targeted attacks.

### Outcome:
- Identified the reconnaissance tool used by the attacker: **Nmap**.

---

## **3. Identifying the Directory Enumeration Tool**

### Tools/Commands:
- **Kibana**: Advanced filtering to track sequential requests and User-Agent strings.

### Steps:
1. Continued filtering logs for `10.0.2.15` and focused on sequential HTTP requests targeting directory paths.
2. Observed patterns like:
   ```
   GET /admin HTTP/1.1
   GET /login HTTP/1.1
   GET /uploads HTTP/1.1
   ```
3. Identified the User-Agent string:
   ```
   Mozilla/5.0 (Gobuster)
   ```
4. Gobuster is a brute-force tool often used for directory enumeration, probing for hidden files and directories on web servers.

### Insights:
- The attacker systematically enumerated directories using **Gobuster**.
- This step is typically used to uncover admin panels, configuration files, or upload endpoints.

### Outcome:
- The attacker used **Gobuster** to map out the server's directory structure.

---

## **4. Counting Failed Resource Requests**

### Tools/Commands:
- **Kibana Query**: Filtered logs for HTTP 404 (Not Found) responses.

### Steps:
1. Filtered logs for the attacker's IP and `404` status codes:
   ```kql
   transaction.remote_address: 10.0.2.15 AND response.status: 404
   ```
2. Reviewed the entries to quantify failed attempts to access non-existent resources.

### Insights:
- The high number of 404 errors (**1,867**) suggested extensive probing by the attacker, testing directories and endpoints.

### Outcome:
- The sheer volume of failed requests highlighted the attacker’s aggressive enumeration strategy.

---

## **5. Discovering the Flag in an Interesting Directory**

### Tools/Commands:
- **Kibana Query**: Focused on successful directory access attempts (`200` status codes).

### Steps:
1. Filtered logs for `200` responses from `10.0.2.15`:
   ```kql
   transaction.remote_address: 10.0.2.15 AND response.status: 200
   ```
2. Analyzed directories accessed during enumeration.
3. Identified a hidden directory containing a flag.

### Insights:
- The flag (`a76637b62ea99acda12f5859313f539a`) demonstrated the attacker’s success in discovering sensitive information.

### Outcome:
- Confirmed the attacker accessed hidden directories successfully.

---

## **6. Discovering the Admin Login Page**

### Tools/Commands:
- **Kibana**: Tracked successful requests to suspicious endpoints.

### Steps:
1. Continued analyzing `200` status responses and noticed repeated access to `/admin-login.php`.
2. Cross-referenced this with Gobuster logs to confirm discovery during enumeration.

### Insights:
- Identifying `/admin-login.php` suggested the attacker now had a specific target for exploitation.

### Outcome:
- The hidden admin login page was identified.

---

## **7. Identifying the Brute-Force Tool**

### Tools/Commands:
- **Kibana Logs**: Focused on login attempts (`POST` requests) and associated User-Agent strings.

### Steps:
1. Filtered logs for `POST` requests to `/admin-login.php` from `10.0.2.15`:
   ```kql
   transaction.remote_address: 10.0.2.15 AND http.method: "POST"
   ```
2. Observed repeated login attempts and the User-Agent string:
   ```
   Mozilla/4.0 (Hydra)
   ```
3. Noted the successful login attempt.

### Insights:
- The attacker used **Hydra**, a brute-force tool, to repeatedly try different credentials.
- Successful login credentials were identified as `admin:thx1138`.

### Outcome:
- Hydra successfully brute-forced the admin login credentials.

---

## **8. Retrieving the Uploaded Web Shell**

### Tools/Commands:
- **Kibana Logs**: Analyzed file uploads (`POST` requests) to `/admin/upload.php`.

### Steps:
1. Filtered logs for requests to `/admin/upload.php`:
   ```kql
   transaction.remote_address: 10.0.2.15 AND http.method: "POST"
   ```
2. Inspected the `request.body` field to identify the contents of the uploaded file.
3. Retrieved a flag embedded within the uploaded web shell:
   ```
   THM{ecb012e53a58818cbd17a924769ec447}
   ```

### Insights:
- The uploaded web shell allowed remote execution of commands on the compromised server.

### Outcome:
- Extracted the flag and confirmed the web shell upload.


## **Lessons Learned**

### 1. **Effective Log Analysis**
   - Tools like Kibana are essential for tracing attacks in real-time and post-event analysis.

### 2. **Reconnaissance Detection**
   - Monitoring unusual User-Agent strings or high request volumes can help detect attacks early.

### 3. **Mitigating Brute Force**
   - Use rate limiting, account lockout policies, and multi-factor authentication to protect login pages.

### 4. **Preventing LFI Attacks**
   - Sanitize user inputs and restrict access to sensitive files to mitigate LFI vulnerabilities.

