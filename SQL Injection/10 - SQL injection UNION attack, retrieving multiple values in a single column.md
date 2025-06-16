# 🧪 Lab 10: Retrieving Multiple Values in a Single Column (SQL Injection) Solution

**PortSwigger Lab Link:** [https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column)

**Table of Contents:**
* 1. Lab Description
* 2. Vulnerability
* 3. Exploitation Steps
* 4. Key Learnings/Concepts
* 5. Mitigation

---

## 1. Lab Description

This lab extends the concept of **UNION-based SQL Injection** to demonstrate how to retrieve **multiple values (like username and password)** and display them within a **single column** on the web page. This is often necessary when the application only prints content from one specific column. The key technique used here is **string concatenation** within the SQL query, which allows combining the output of multiple database fields into one.

---

## 2. Vulnerability

The vulnerability remains **SQL Injection**, specifically a **UNION-based SQL Injection** allowing for **data exfiltration**. The application's failure to properly sanitize user input before constructing SQL queries permits an attacker to append a `UNION SELECT` statement.

While previous labs focused on finding columns that display *any* string, this lab addresses a common scenario where only one column's content is actually rendered to the user. To bypass this limitation and extract multiple pieces of information (e.g., both username and password), attackers leverage database-specific string concatenation functions within their injected `UNION SELECT` query. This allows them to combine several pieces of data into a single string that can then be displayed by the application.

---

## 3. Exploitation Steps

### 🔹 Step 1: Intercept Request with Burp Suite

1.  Open the lab in your web browser.
2.  Ensure your browser is configured to proxy traffic through **Burp Suite**.
3.  Interact with the application (e.g., applying a product filter or search) to generate an HTTP request.
4.  In Burp Suite's "Proxy" tab, intercept this request and send it to **Repeater** for easier manipulation.

### 🔹 Step 2: Determine Column Count

First, establish the number of columns the original backend query returns. This can be done using the `ORDER BY` clause or by testing `UNION SELECT NULL` payloads.

1.  In Burp Repeater, identify the vulnerable parameter.
2.  Test with `ORDER BY 1--`, `ORDER BY 2--`, etc., until an error, or by gradually adding `NULL`s to a `UNION SELECT` payload (`' UNION SELECT NULL--`, `' UNION SELECT NULL, NULL--`, etc.) until an `HTTP/200 OK` response is received.

* 📌 **In this lab, we found that the query returns 2 columns.**

### 🔹 Step 3: Identify String-Compatible Columns

Next, determine which of the identified columns can successfully display string data. This is crucial as we'll be concatenating strings.

1.  Test by injecting a string into each column position, one at a time:
    * Try: `' UNION SELECT 'hello', NULL--`
    * Observe the response. If it's an error (e.g., `500`), the first column is likely an integer.
    * Try: `' UNION SELECT NULL, 'hello'--`
    * Observe the response.

* 📌 **In this lab, we found that the first column expects an integer, and the second column is string-compatible.** (So, `' UNION SELECT 'hello', 'hello'--` resulted in a `500 error`, while `' UNION SELECT NULL, 'hello'--` resulted in a `200 OK` and printed 'hello').

### 🔹 Step 4: Retrieve Username and Password Using Concatenation

Since we have two columns, one of which can display strings, and we want to retrieve both the username and password (which are strings) within that single displayable column, we'll use a database-specific string concatenation operator. The double pipe `||` is a common concatenation operator in SQL (especially Oracle/PostgreSQL).

1.  Craft the final `UNION SELECT` payload to concatenate `username` and `password` from the `users` table into the string-compatible column. We'll use a separator (like `~` or `:`) to distinguish them. The integer column will be filled with a placeholder integer (e.g., `1`).

    ```sql
    ' UNION SELECT 1, username||'~'||password FROM users--
    ```
    * *Explanation:*
        * `1`: Fills the first (integer) column.
        * `username||'~'||password`: Concatenates the `username`, a separator (`~`), and the `password` into a single string.
        * `FROM users`: Specifies the table from which to retrieve the data.
        * `--`: Comments out the rest of the original query.

2.  **Important:** URL encode the payload if you are inserting it directly into the URL path or query parameter. Burp's "URL-encode key characters" function is very helpful for this.

3.  Insert this payload into the vulnerable parameter in Burp Repeater and send the request.

4.  In the Response tab, look for the concatenated username and password displayed on the page (e.g., `administrator~abcdef123`). This allows you to clearly separate them.

5.  Copy the administrator's username and password.

### 🔹 Step 5: Log In and Solve the Lab

1.  Navigate to the application's login page.
2.  Use the extracted administrator username and password to log in.
3.  Upon successful login, the lab will be marked as solved.

---

## 4. Key Learnings/Concepts

* **String Concatenation in SQL:** Understanding how to combine multiple database fields into a single string output. Common operators/functions include:
    * `||` (Oracle, PostgreSQL, SQLite)
    * `+` (Microsoft SQL Server)
    * `CONCAT()` (MySQL, PostgreSQL)
    * `CONCAT_WS()` (MySQL)
* **Overcoming Output Limitations:** This lab teaches a crucial technique for scenarios where only one column's content is displayed by the application, allowing full data exfiltration even under such constraints.
* **Data Type Awareness:** Reinforces the importance of knowing which columns are string-compatible to ensure the concatenated string is successfully displayed.
* **Choosing Separators:** The use of a distinct separator (like `~` or `|||`) is vital to easily parse the extracted data (e.g., `username~password` vs. `usernamepassword`).
* **Advanced Data Exfiltration:** This is a more advanced technique in the data exfiltration phase of SQL Injection, making it possible to retrieve all desired information efficiently.

---

## 5. Mitigation

To prevent this type of SQL Injection and unauthorized data retrieval, implement the following robust security measures:

* **Parameterized Queries (Prepared Statements):** This is the most effective defense. By using placeholders for all user input, the database engine processes user input strictly as data values, preventing any malicious SQL (including concatenation operators) from being executed as commands.
* **Strong Input Validation:** Validate all user inputs against strict whitelist rules (e.g., expected data types, formats, lengths). Rejecting malformed input at the application layer helps reduce the attack surface.
* **Least Privilege Principle:** Database accounts used by the application should operate with the absolute minimum permissions required for their specific functions. They should not have access to sensitive tables (like `users`) or unnecessary system views.
* **Secure Error Handling:** Avoid displaying detailed database error messages to users, as these can provide critical clues about database structure, data types, and potential vulnerabilities. Log errors securely on the server side and present only generic messages to the user.
* **Web Application Firewalls (WAFs):** While not a primary defense, a WAF can provide an additional layer of defense by detecting and blocking common SQL injection attack patterns, although it should not be relied upon as the sole defense.
* **Database Hardening:** Regularly patch and securely configure the database server. Disable unnecessary features, services, and default accounts.
