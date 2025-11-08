# 🧪 Lab 15: Blind SQL Injection with Time Delays (PostgreSQL) Solution

**PortSwigger Lab Link:** https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval

**Table of Contents:**
* 1. Lab Description & Goal
* 2. Vulnerability: Conditional Time Delay Oracle
* 3. Step-by-Step Exploitation (Burp Intruder)
* 4. Optimization: Burp Intruder Resource Pool
* 5. Key Learnings/Concepts
* 6. Mitigation

---

## 1. Lab Description & Goal

This lab performs **Time-based Blind SQL Injection** to retrieve sensitive information. Since the application is running on a **PostgreSQL** database (indicated by the `PG_SLEEP` function), the exploitation relies entirely on measuring the server's response time to determine if an injected condition is true or false.

* **Goal:** Extract the administrator's password and log in.
* **Injection Point:** The **`TrackingId` cookie**.
* **The Oracle:** **Response Time** ($\approx 10$ seconds for TRUE, $< 1$ second for FALSE).

## 2. Vulnerability: Conditional Time Delay Oracle

We use a **conditional time delay** payload to create the TRUE/FALSE oracle. This requires the `CASE WHEN` statement to execute the time-delay function (`PG_SLEEP(10)`) only if our guessed condition is met.

### A. Confirmation Test (in Burp Repeater)

1.  **Test 1 (Always TRUE $\implies$ Delay):**
    ```sql
    ' || SELECT CASE WHEN (1=1) THEN PG_SLEEP(10) ELSE PG_SLEEP(0) END || '
    ```
    * **Result:** Server response takes **$\approx 10$ seconds**. This confirms the query is working and the database is vulnerable.

2.  **Test 2 (Always FALSE $\implies$ No Delay):**
    ```sql
    ' || SELECT CASE WHEN (1=2) THEN PG_SLEEP(10) ELSE PG_SLEEP(0) END || '
    ```
    * **Result:** Server response is **immediate** (< 1 second).

This confirms our precise, conditional time oracle.

## 3. Step-by-Step Exploitation (Burp Intruder)

### A. Step 1: Find Password Length (Sniper Attack)

We use the `LENGTH()` function to find the length by checking if the length is greater than the number we are guessing (`> N`).

1.  **Length Payload:**
    ```sql
    ' || SELECT CASE WHEN (username='administrator' AND LENGTH(password) > §N§) THEN PG_SLEEP(10) ELSE PG_SLEEP(0) END FROM users--
    ```
    (Note: The `FROM users` is often needed for the subquery to execute correctly).
2.  **Burp Intruder Setup (Sniper):**
    * Attack Type: **Sniper**.
    * Payload Position: The number `N` in the payload (using `§N§`).
    * **Payload Set 1:** Type: **Numbers**, From: `1`, To: `30` (or reasonable limit).
3.  **Result Analysis:**
    * Start the attack. In the results section, go to **Columns** and select **Response Received** time.
    * As long as the condition (`LENGTH > N`) is TRUE, the response time will be $\approx 10$ seconds.
    * Find the request number where the time **drops drastically** (from $\approx 10$s to $< 1$s).
    * If the time drops at $\mathbf{N=20}$, the length is $20$ (since $20 > 19$ is TRUE, but $20 > 20$ is FALSE).
    * **Conclusion:** The password length is **20**.

### B. Step 2: Extract Password Characters (Cluster Bomb Attack)

We use the `SUBSTRING()` function to guess characters one by one, checking if the guessed character is correct.

1.  **Character Extraction Payload (2 Positions):**
    ```sql
    ' || SELECT CASE WHEN (username='administrator' AND SUBSTRING(password, §I§, 1) = '§C§') THEN PG_SLEEP(10) ELSE PG_SLEEP(0) END FROM users--
    ```
    (Where `§I§` is the position and `§C§` is the character guess).
2.  **Burp Intruder Setup (Cluster Bomb):**
    * Attack Type: **Cluster Bomb**.
    * **Payload Set 1 (Position $\S I \S$):** Type: **Numbers**, From: `1`, To: `20`, Step: `1`.
    * **Payload Set 2 (Character $\S C \S$):** Type: **Brute Forcer** (or List of characters), Min/Max Length: `1`.
3.  **Result Analysis:**
    * Sort the results by the **Response Time** column (descending).
    * The combinations of `(I, C)` that resulted in the **biggest response time ($\approx 10$ seconds)** are the correct character guesses.
4.  **Solve:** Reconstruct the password and log in as `administrator`.

## 4. Optimization: Burp Intruder Resource Pool

Time-based attacks are very slow. To run the Cluster Bomb attack efficiently, it is crucial to manage Burp's concurrent requests.

1.  Go to **Intruder $\rightarrow$ Options $\rightarrow$ Resource Pool**.
2.  **Create a New Resource Pool.**
3.  Set the **Maximum concurrent requests** to a low value, typically **1**.
    * **Reason:** If Burp sends multiple requests concurrently (the default is 10), the response times will overlap, making it impossible to distinguish which payload caused the 10-second delay. Setting it to 1 ensures each request is measured accurately.

---

## 5. Key Learnings/Concepts

* **Time-Based Blind SQLi:** The most challenging form of blind injection; it extracts data based solely on **response latency** .
* **Conditional SLEEP/DELAY:** Requires injecting the delay function inside a conditional statement (`CASE WHEN`) to make the delay dependent on a true condition.
* **PostgreSQL Payload:** The time-delay function is `PG_SLEEP(N)`.
* **Time Measurement Oracle:** Unlike Boolean (text match) or Error-based (status code), the Time-based oracle requires sorting and analyzing the **Response Time
