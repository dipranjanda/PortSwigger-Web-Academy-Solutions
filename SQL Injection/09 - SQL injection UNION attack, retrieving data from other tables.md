# 🧪 Lab 9: Retrieving Data from Other Tables (SQL Injection) Solution

**PortSwigger Lab Link:** https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables

**Table of Contents:**
* 1. Lab Description
* 2. Vulnerability
* 3. Exploitation Steps
* 4. Key Learnings/Concepts
* 5. Mitigation

---

## 1. Lab Description

This lab is a culmination of previous SQL Injection techniques. The objective is to exploit a **UNION-based SQL Injection vulnerability** to retrieve sensitive data, specifically the **administrator's username and password**, from a hidden table named `users`. This requires identifying the correct number of columns, finding string-compatible columns, and then constructing a `UNION SELECT` query to extract the desired credentials.

---

## 2. Vulnerability

The vulnerability is **SQL Injection**, specifically a **UNION-based SQL Injection** that allows for **data exfiltration**. The application constructs its SQL queries dynamically using user-supplied input without proper sanitization. This flaw enables an attacker to append a `UNION SELECT` statement to the original query, combining the results from a different, potentially hidden, table.

This lab leverages the knowledge gained from previous labs: identifying the number of columns and determining which of those columns can successfully display string data. With this information, an attacker can precisely craft a query to select sensitive information (like usernames and passwords) from any accessible table in the database and have it displayed in the application's response.

---

## 3. Exploitation Steps

### 🔹 Step 1: Intercept Request with Burp Suite

1.  Open the lab in your web browser.
2.  Ensure your browser is configured to proxy traffic through **Burp Suite**.
3.  Generate an HTTP request by interacting with the application (e.g., applying a filter or searching).
4.  In Burp Suite's "Proxy" tab, intercept this request and send it to **Repeater** for easier manipulation.

### 🔹 Step 2: Determine Column Count

First, we need to know how many columns the original backend query is returning. We can use the `ORDER BY` technique or iteratively try `UNION SELECT NULL` payloads.

1.  In Burp Repeater, identify the vulnerable parameter.
2.  Test with `ORDER BY 1--`, `ORDER BY 2--`, etc., until an error occurs. The last successful number indicates the column count.
    * Alternatively, use `' UNION SELECT NULL--`, `' UNION SELECT NULL, NULL--`, etc., until you get an `HTTP/200 OK` response.

* 📌 **In this lab, we found that the query returns 2 columns.**

### 🔹 Step 3: Find String-Compatible Columns

Next, we need to identify which of the two columns can display string data. This is crucial because our target data (username and password) are strings.

1.  Test by injecting a string into each column position, one at a time, using `NULL` for the other.
    * Try: `' UNION SELECT 'hello', NULL--`
    * Observe the response.
    * Try: `' UNION SELECT NULL, 'hello'--`
    * Observe the response.

* 📌 **In this lab, we found that both columns are capable of supporting string data.** This simplifies the next step, as we can place `username` and `password` in any order.

### 🔹 Step 4: Retrieve Username and Password from the `users` Table

Now that we know the column count (2) and that both columns support strings, we can directly select the `username` and `password` columns from the `users` table.

1.  Craft the final `UNION SELECT` payload to retrieve the desired data. Based on common database naming conventions for user tables, we will assume a table named `users` with columns `username` and `password`.
    ```sql
    ' UNION SELECT username, password FROM users--
    ```
    * Remember to URL encode the payload if necessary (Burp Suite's "URL-encode key characters" function is useful).
2.  Insert this payload into the vulnerable parameter in Burp Repeater and send the request.
3.  In the Response tab of Burp Repeater, carefully examine the HTML body.

4.  📌 **You should see the username and password (including the administrator's credentials) displayed on the page.** Copy these credentials.

### 🔹 Step 5: Log In and Solve the Lab

1.  Navigate to the application's login page.
2.  Use the extracted administrator username and password to log in.
3.  Upon successful login, the lab will be marked as solved.

---

## 4. Key Learnings/Concepts

* **Full SQLi Exploitation Flow:** This lab demonstrates a complete end-to-end SQL Injection attack, from initial reconnaissance (column count and string-compatible columns) to data extraction.
* **Targeted Data Retrieval:** Understanding how to select specific columns (`username`, `password`) from a known table (`users`) using `UNION SELECT`.
* **Database Naming Conventions:** Recognizing common table and column names (`users`, `username`, `password`) can speed up exploitation, though in real-world scenarios, these would be enumerated using techniques like those in Lab 4 (Oracle `all_tables`/`all_tab_columns`) or similar methods for other databases.
* **Practical Application of `UNION SELECT`:** This lab shows a direct, practical application of `UNION SELECT` for extracting valuable sensitive information.
* **Reconnaissance Integration:** Emphasizes that successful data exfiltration often relies on the preliminary steps of determining column count and data type compatibility.

---

## 5. Mitigation

Preventing this type of data exfiltration via SQL Injection is critical and relies on robust security practices:

* **Parameterized Queries (Prepared Statements):** This is the most effective defense. By using placeholders for all user input, the database engine distinguishes between executable code and data, preventing any injected SQL from being processed as a command.
* **Strong Input Validation:** Validate all user inputs against strict whitelist rules (e.g., expected data types, formats, lengths). Rejecting malformed input at the application layer reduces the attack surface.
* **Least Privilege Principle:** Database accounts used by the application should have only the minimum necessary permissions to perform their functions. They should not have access to sensitive tables (like `users` tables for all users) or system views unless strictly required.
* **Secure Error Handling:** Avoid displaying detailed database error messages to users, as these can provide clues about database structure, table names, and column names. Log errors securely on the server side and present generic messages to the user.
* **Web Application Firewalls (WAFs):** While not a primary defense, a WAF can help detect and block common SQL injection attack patterns, providing an additional layer of security and logging capabilities.
* **Database Hardening:** Regularly patch and configure the database server securely, including disabling unnecessary features and services.
