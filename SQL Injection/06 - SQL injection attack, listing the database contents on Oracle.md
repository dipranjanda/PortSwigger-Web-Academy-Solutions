# 🧪 Lab: Listing the Database Contents on Oracle (SQL Injection) Solution

**PortSwigger Lab Link:** [You'll insert the direct link to the PortSwigger Lab here when you upload it to GitHub]

**Table of Contents:**
* 1. Lab Description
* 2. Vulnerability
* 3. Exploitation Steps
* 4. Key Learnings/Concepts
* 5. Mitigation

---

## 1. Lab Description

This lab presents a classic **SQL Injection vulnerability** in an application running on an **Oracle database**. The goal is to exploit this vulnerability to **dump the administrator's password** from the database and log in. This involves identifying the number of columns, listing database tables, finding relevant columns (like username and password), and finally extracting the credentials.

---

## 2. Vulnerability

The vulnerability in this lab is **SQL Injection**, specifically a **UNION-based SQL Injection** that allows for **database enumeration**. This occurs because the application uses unsanitized user input directly in its SQL queries. An attacker can inject malicious SQL code, such as `UNION SELECT` statements, to combine the results of their own query with the application's legitimate query.

In an Oracle database, we can leverage built-in data dictionary views (like `all_tables` and `all_tab_columns`) to discover table names and column names. This is a common technique for enumerating the database schema when SQL Injection is present. Once the schema is understood, sensitive data like user credentials can often be extracted.

---

## 3. Exploitation Steps

### 🔹 Step 1: Intercept Request with Burp Suite

1.  Open the lab in your web browser.
2.  Ensure your browser is configured to send traffic through **Burp Suite**.
3.  Navigate to a page or perform an action (e.g., filter products, search) that generates an HTTP request containing a parameter you suspect might be vulnerable to SQL injection.
4.  In Burp Suite's "Proxy" tab, intercept this request and send it to **Repeater** for easier manipulation and testing.

### 🔹 Step 2: Determine Number of Columns with `ORDER BY`

Before using `UNION SELECT`, we need to find out how many columns the original SQL query returns. This is crucial for our `UNION SELECT` statement to have the correct number of columns.

1.  In Burp Repeater, identify the vulnerable parameter (e.g., `category` or `id`).
2.  Inject `ORDER BY` clauses incrementally into the parameter, appending `--` to comment out the rest of the original query:
    * `' ORDER BY 1--`
    * `' ORDER BY 2--`
    * `' ORDER BY 3--`
3.  Continue increasing the number until you receive an SQL error from the application (e.g., "invalid column number").
4.  The last successful number *before* the error tells you the exact number of columns the query returns.

* 📌 **In this lab, we found that the query returns 2 columns.**

### 🔹 Step 3: List Database Tables (Oracle)

Now that we know the column count (2), we can use `UNION SELECT` to query Oracle's data dictionary views to list table names. The `all_tables` view contains information about all tables accessible to the current user.

1.  Use the following `UNION SELECT` payload to retrieve table names:
    ```sql
    ' UNION SELECT table_name, NULL FROM all_tables--
    ```
    * *Note:* We use `NULL` for the second column to match the two columns identified earlier.
2.  Insert this payload into the vulnerable parameter in Burp Repeater and send the request.
3.  In the response, look for table names. You might need to scroll through the response or use Burp's search function (Ctrl+F or Cmd+F).
4.  📌 **We are looking for a table that likely contains user credentials. In this lab, we found a promising table named `USERS_RQNPQ`.**

### 🔹 Step 4: List Columns from the Target Table

After identifying a potential table (`USERS_RQNPQ`), the next step is to find out what columns it contains. Oracle's `all_tab_columns` view stores information about columns in all tables.

1.  Use the following `UNION SELECT` payload to retrieve column names from `USERS_RQNPQ`:
    ```sql
    ' UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name = 'USERS_RQNPQ'--
    ```
    * *Important:* The table name `'USERS_RQNPQ'` inside the `WHERE` clause needs to be in single quotes.
2.  Insert this payload into the vulnerable parameter and send the request.
3.  In the response, search for column names.
4.  📌 **We found two relevant columns: `USERNAME_GYPT` (for usernames) and `PASSWORD_GGVR` (for passwords).**

### 🔹 Step 5: Dump Administrator Credentials

Finally, with the table name (`USERS_RQNPQ`) and column names (`USERNAME_GYPT`, `PASSWORD_GGVR`) identified, we can construct a `UNION SELECT` query to extract the actual administrator's username and password.

1.  Use the following `UNION SELECT` payload to retrieve the username and password:
    ```sql
    ' UNION SELECT USERNAME_GYPT, PASSWORD_GGVR FROM USERS_RQNPQ--
    ```
2.  Insert this payload into the vulnerable parameter and send the request.
3.  In the response, you should now see the administrator's username and password displayed on the page.
4.  Copy these credentials.

### 🔹 Step 6: Log In and Solve the Lab

1.  Go to the application's login page.
2.  Use the extracted administrator username and password to log in.
3.  If successful, the lab will be marked as solved!

---

## 4. Key Learnings/Concepts

* **Oracle SQL Injection:** Understanding how `UNION SELECT` attacks can be performed specifically on Oracle databases.
* **Database Schema Enumeration:** Learning how to discover table names and column names within an Oracle database using built-in data dictionary views (`all_tables`, `all_tab_columns`). This is a crucial step in many SQL injection scenarios where direct table/column names are not known.
* **`ORDER BY` for Column Counting:** Reinforces the foundational technique to determine the number of columns in the original query.
* **`UNION SELECT` for Data Retrieval:** Demonstrates how `UNION SELECT` is used to extract arbitrary data, including sensitive information like credentials, from different tables.
* **Oracle Data Dictionary Views:**
    * `all_tables`: Provides information about all tables the current user has access to.
    * `all_tab_columns`: Provides information about all columns in all tables the current user has access to.
* **Targeted Data Extraction:** Once table and column names are known, an attacker can precisely target and extract specific data.
* **Burp Suite Repeater:** Its iterative request modification and response inspection capabilities are essential for performing these multi-step attacks.

---

## 5. Mitigation

Preventing SQL Injection, especially against database enumeration, requires robust coding practices:

* **Parameterized Queries (Prepared Statements):** This is the single most effective defense. Instead of building SQL queries by concatenating strings, use prepared statements where placeholders are used for user-supplied data. The database engine then treats the input purely as data, not as executable code.
* **Input Validation:** Implement strict validation on all user inputs. While not a standalone solution, it acts as an important secondary defense by rejecting inputs that don't conform to expected formats.
* **Least Privilege Principle:** Configure database users with the absolute minimum privileges required for their operations. This limits what an attacker can achieve even if an SQL injection vulnerability is exploited (e.g., preventing access to sensitive system tables like `all_tables`).
* **Web Application Firewalls (WAFs):** A WAF can detect and block known SQL injection attack patterns. While not a primary defense, it provides an additional layer of security and can alert to ongoing attacks.
* **Secure Error Handling:** Avoid displaying detailed database error messages to users. Such errors can provide attackers with valuable information about the database structure and backend. Instead, log errors securely and present generic error messages to the user.
