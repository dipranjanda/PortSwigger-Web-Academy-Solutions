# 🧪 Lab 14: Blind SQL Injection with Time Delays Solution

**PortSwigger Lab Link:** https://portswigger.net/web-security/sql-injection/blind/lab-time-delays

**Table of Contents:**
* 1. Lab Description & Goal
* 2. Vulnerability: Time-Based Oracle
* 3. Core Concept: Conditional Time Delay
* 4. Step-by-Step Exploitation (Time Delay Confirmation)
* 5. Key Learnings/Concepts
* 6. Mitigation

---

## 1. Lab Description & Goal

The objective of this lab is to demonstrate and confirm the existence of a **Time-based Blind SQL Injection** vulnerability.

* **Goal:** Successfully exploit the SQL vulnerability to **cause a noticeable time delay** (e.g., 10 seconds) in the application's response.
* **Vulnerability Location:** The injection point is in the **`TrackingId` cookie**, requiring string concatenation to append our malicious query.

---

## 2. Vulnerability: Time-Based Oracle

**Time-based Blind SQLi** is the most covert and difficult type of Blind SQLi. It is used when neither direct data output (UNION) nor an application change (Boolean response) is available.

* **The Oracle:** The success of the injection is determined by measuring the **time** it takes for the server to respond.
* **Mechanism:** An attacker injects a conditional payload that tells the database: "If this condition is TRUE, execute a time-consuming function (like `SLEEP(10)`); otherwise, execute nothing."
    * **TRUE Condition** (Correct Guess) $\implies$ Server executes `SLEEP(10)` $\implies$ **Response time is ~10 seconds.**
    * **FALSE Condition** (Incorrect Guess) $\implies$ Server bypasses `SLEEP(10)` $\implies$ **Response time is immediate (< 1 second).**

## 3. Core Concept: Conditional Time Delay

The specific time-delay function varies by database type. Since the payload notes a `PG_SLEEP`, we assume the database is **PostgreSQL**.

### PostgreSQL Time Delay Payload

To test the delay without a condition, we use the following structure in the `TrackingId` cookie:

| Database | Delay Function | Example Payload |
| :--- | :--- | :--- |
| **PostgreSQL** | `PG_SLEEP(N)` | `xyz' || PG_SLEEP(10)--` |

* **Your Payload:** `' || PG_SLEEP(10) || '`
    * The `||` operator is used for **string concatenation** in PostgreSQL (and Oracle), ensuring the `PG_SLEEP` function is successfully appended to the original query within the `TrackingId` cookie.

---

## 4. Step-by-Step Exploitation (Time Delay Confirmation)

The first step in any Time-based SQLi is confirming that the delay works.

### 🔹 Step 1: Intercept Request and Identify Injection Point

1.  Open **Burp Suite** and ensure it's intercepting traffic.
2.  Navigate the lab to capture a request that includes the **`TrackingId`** cookie.
3.  Send the request to **Repeater**.

### 🔹 Step 2: Establish a Baseline Time

1.  In Repeater, send the original request (without any changes) several times.
2.  Note the typical response time (usually less than 50 milliseconds). This is your **baseline time**.

### 🔹 Step 3: Inject the Delay Payload

1.  In the `TrackingId` cookie, replace the value with your injection, ensuring proper syntax for concatenation. Assume the original cookie value is `abc`.
2.  **Injected `TrackingId`:**
    ```
    abc' || PG_SLEEP(10)--
    ```
    * `abc'` : Closes the string quote used by the backend query.
    * `||`: Concatenates the original string with the `PG_SLEEP` function call.
    * `PG_SLEEP(10)`: Instructs the PostgreSQL server to pause query execution for 10 seconds.
    * `--`: Comments out the rest of the original query.

3.  **URL Encoding Note:** Ensure the single quote `'` and the space before the comment `--` are correctly encoded if the browser or Burp doesn't handle it automatically (e.g., `'` becomes `%27`, space becomes `%20`).

### 🔹 Step 4: Measure the Response Time

1.  In Burp Repeater, send the request containing the delay payload.
2.  **Measure the Response Time** (visible in the top-right corner of the Repeater panel).

* **Successful Result:** The response time should be **approximately 10,000 milliseconds (10 seconds)**.

This successful delay confirms the vulnerability and the database type (PostgreSQL in this case), allowing you to proceed with conditional time-based data extraction in subsequent labs.

---

## 5. Key Learnings/Concepts

* **Time-Based Blind SQLi:** A method of inferring data based on the **server's response time**, used when other channels (output, errors, Boolean responses) are unavailable.
* **Conditional Attack:** The full attack uses the sleep function *only* if a guessed condition is true (e.g., `AND 1=1`), making it a binary (true/false) oracle.
* **Database-Specific Payloads:** Time-based injection relies on unique database functions:
    * **PostgreSQL:** `PG_SLEEP(N)`
    * **MySQL:** `SLEEP(N)`
    * **Microsoft SQL Server:** `WAITFOR DELAY '0:0:N'`
* **String Concatenation (`||`):** Required here to break out of the original SQL query string and append the new function call, particularly common in PostgreSQL and Oracle.
* **Automation is Mandatory:** Time-based attacks are too slow for manual testing. **Burp Intruder** (Grep-Time setting) or **Python scripts** are essential for automation.

---

## 6. Mitigation

Preventing Time-based Blind SQL Injection is achieved by eliminating the core vulnerability:

* **Parameterized Queries (Prepared Statements):** This is the fundamental defense. It ensures all user input, including the `TrackingId` cookie content, is treated as data, preventing SQL functions like `PG_SLEEP()` from being executed.
* **Input Validation:** Strictly validate all inputs, including cookie values. Reject any input that contains SQL keywords, functions (`SLEEP`, `DELAY`, `SELECT`), or special characters used for injection (`'`, `||`, `--`).
* **Database Least Privilege:** The application's database user should not have permissions to execute non-essential functions like time delay or system calls, limiting the attacker's tools even if an injection is possible.
* **Monitoring and Throttling:** Implement monitoring to detect and throttle clients that send a high number of requests resulting in unusual, consistent time delays, which is characteristic of a time-based attack.
