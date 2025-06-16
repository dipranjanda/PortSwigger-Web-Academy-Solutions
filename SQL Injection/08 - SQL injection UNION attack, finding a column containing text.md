# 🧪 Lab 8: Finding a Column Containing Text (SQL Injection) Solution

**Portswigger Lab Link:** [You'll insert the direct link to the PortSwigger Lab here when you upload it to GitHub]

**Table of Contents:**
* 1. Lab Description
* 2. Vulnerability
* 3. Exploitation Steps
* 4. Key Learnings/Concepts
* 5. Mitigation

---

## 1. Lab Description

This lab is a crucial follow-up to determining the column count in SQL Injection. The objective is to identify which of the backend query's columns can display **string data** when performing a `UNION SELECT` attack. This is essential because `UNION SELECT` requires not only a matching number of columns but also compatible data types. By successfully injecting a string into one of the columns and seeing it reflected on the page, we confirm its compatibility for future data extraction.

---

## 2. Vulnerability

The vulnerability is still **SQL Injection**, specifically utilizing **UNION-based SQL Injection**. The application processes user input without proper validation or sanitization, allowing an attacker to append a `UNION SELECT` statement to the original query.

This lab highlights a key constraint of `UNION SELECT`: for the union to succeed, the data types of corresponding columns in both the original query and the injected `SELECT` statement must be compatible. If a column in the original query expects an integer, and you try to select a string into it via `UNION`, it will result in a database error. This lab focuses on systematically identifying columns that can accept string data, which is vital for extracting text-based information (like usernames, passwords, or error messages) in later exploitation stages.

---

## 3. Exploitation Steps

### 🔹 Step 1: Intercept Request and Confirm Column Count

1.  Open the lab in your web browser and ensure Burp Suite is intercepting traffic.
2.  Navigate to a page that likely involves a database query and send the request to **Repeater**.
3.  **From previous labs (or by testing with `ORDER BY` or `UNION SELECT NULL` payloads as in Lab 7), confirm the number of columns returned by the backend query.**

    * 📌 **In this lab, we know from our previous work that the query returns 3 columns.**

### 🔹 Step 2: Attempt `UNION SELECT` with a String Payload

Now that we know the column count, we will try to inject a string into one of the columns using `UNION SELECT`. We'll start by placing the string in the first position and filling the other columns with `NULL`.

1.  In Burp Repeater, craft the `UNION SELECT` payload to inject a test string (e.g., `'ABCDE'`) into the first column, with `NULL`s for the remaining columns:
    ```sql
    ' UNION SELECT 'ABCDE', NULL, NULL--
    ```
    * Remember to URL encode the payload if you are inserting it directly into the URL path or query parameter (e.g., `'` becomes `%27`, spaces `%20`, etc. – Burp's URL-encode function is helpful).
2.  Send the request.

3.  **Observe the response:**
    * 📌 **In this lab, sending `' UNION SELECT 'ABCDE', NULL, NULL--` resulted in an `HTTP/500 Internal Server Error`.** This indicates that the first column is *not* compatible with string data (it likely expects an integer or other non-string data type).

### 🔹 Step 3: Find the String-Compatible Column by Shifting the Payload

Since the first column didn't accept the string, we need to systematically try injecting the string into the other column positions until we find one that accepts it without an error. This is because `UNION SELECT` requires data type compatibility between corresponding columns.

1.  Modify the `UNION SELECT` payload to place the test string in the **second column**, keeping `NULL`s for others:
    ```sql
    ' UNION SELECT NULL, 'ABCDE', NULL--
    ```
2.  Send the request.

3.  **Observe the response:**
    * If this also gives an error, the second column is also not string-compatible.

4.  Modify the `UNION SELECT` payload to place the test string in the **third column**:
    ```sql
    ' UNION SELECT NULL, NULL, 'ABCDE'--
    ```
5.  Send the request.

6.  **Observe the response:**
    * 📌 **In this lab, we find that when the payload `' UNION SELECT NULL, NULL, 'ABCDE'--` is sent, the application returns an `HTTP/200 OK` response, and the string "ABCDE" is displayed on the web page.**

This confirms that the third column is the one compatible with string data and is being outputted to the page.

### 🔹 Step 4: Lab Solved!

Once the string `ABCDE` (or your chosen test string) is successfully displayed on the page, the lab will be marked as solved. This knowledge is crucial for future data extraction efforts, as you now know which column to use to retrieve text-based information like usernames or passwords.

---

## 4. Key Learnings/Concepts

* **Data Type Compatibility in `UNION SELECT`:** This is the primary learning point. For `UNION SELECT` to work, not only must the number of columns match, but the data types of corresponding columns must also be compatible. Attempting to select a string into an integer-only column will cause an error.
* **Column Identification for String Output:** Demonstrates a systematic approach to find which column(s) are actually being displayed on the web page and are capable of rendering string data. This is vital for extracting text-based sensitive information later.
* **Iterative Testing:** Reinforces the importance of iterative testing with Burp Repeater, adjusting payloads based on observed error messages or successful responses.
* **`NULL` as a Placeholder:** Confirms the utility of `NULL` as a universal placeholder for columns where the data type is unknown or irrelevant to the immediate goal.
* **SQL Injection Reconnaissance:** This lab represents another essential step in the reconnaissance phase of SQL injection, moving beyond just column counting to understanding data type output capabilities.

---

## 5. Mitigation

To prevent vulnerabilities like the one exploited in this lab, the core security practices for SQL Injection remain paramount:

* **Parameterized Queries (Prepared Statements):** The most effective defense. These separate SQL code from user-supplied data, ensuring that string inputs (even malicious ones) are treated as data values and not as executable SQL commands, thus preventing data type mismatches from being exploitable.
* **Strong Input
