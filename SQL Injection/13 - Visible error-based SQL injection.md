# 🧪 Lab 13: Visible Error-Based SQL Injection Solution

**PortSwigger Lab Link:**  https://portswigger.net/web-security/sql-injection/blind/lab-sql-injection-visible-error-based

**Table of Contents:**
* 1. Lab Description & Goal
* 2. Vulnerability: Error-Based Oracle
* 3. Exploitation Steps: Dumping Data via Error
* 4. Overcoming Query Restrictions (LIMIT & OFFSET)
* 5. Key Learnings/Concepts
* 6. Mitigation

---

## 1. Lab Description & Goal

This lab exploits a **Visible Error-Based SQL Injection** vulnerability. Unlike Blind SQLi, the server is configured to **print the actual database error message** in the application's response, leading to **information disclosure**.

* **Goal:** Dump the administrator's password and log in by forcing the database to include the password within a forced error message.
* **Vulnerability Location:** The injection point is likely in the **`TrackingId` cookie** (as in previous Blind labs).
* **The Oracle:** The **Error Message** itself serves as the output channel for the data.

## 2. Vulnerability: Error-Based Oracle

The key to error-based SQLi is to attempt a mathematical operation on a non-numeric string (the data we want to retrieve). This forces the database to generate an error that contains the sensitive string.

### Forcing the Error

The traditional method is to force a division-by-zero error (`1/0`) on a string. Since division is only possible on integers, the database attempts to cast the string (our password) into an integer, fails, and prints the string in the error message.

In this specific lab, we use the `CAST()` function, a powerful tool for explicit data type conversion. We attempt to convert a **string** (the password) into a non-compatible data type like **Integer** (`INT`) or **Boolean** (`BOOL`).

## 3. Exploitation Steps: Dumping Data via Error

### A. Initial Attempt with CAST (Long Query)

1.  In Burp Repeater, inject the following payload into the `TrackingId` cookie (assuming the base query involves an `AND` statement):
    ```sql
    ' AND CAST((SELECT password FROM users WHERE username='administrator') AS INT)--
    ```
2.  **Result:** This attempt fails, often due to a **character restriction** or the query being too long for the injection context.

### B. Shortening the Query (LIMIT Trick)

Since we have a restriction on query length, we must shorten the statement. The full sub-query `(SELECT password FROM users WHERE username='administrator')` can be shortened, as the target table is often implied, and we only need one result at a time.

1.  Use the `LIMIT 1` function to tell the database to only return one result from the `users` table, which is often sufficient to trigger the error.
    ```sql
    ' AND CAST((SELECT password FROM users LIMIT 1) AS INT)--
    ```
2.  **Result:** This is shorter, but still results in an error stating that the `AND` condition expects a **Boolean** (True/False) value, not an **Integer** (`INT`).

### C. Final Working Payload

1.  We change the target cast function from `INT` to **`BOOL` (Boolean)** to satisfy the requirement of the `AND` clause.
2.  **Final Payload:**
    ```sql
    ' AND CAST((SELECT password FROM users LIMIT 1) AS BOOL)--
    ```
3.  **Result:** The database attempts to convert the string (the first password) into a Boolean, fails, and **prints the first password in the database within the visible error message**.

## 4. Overcoming Query Restrictions (LIMIT & OFFSET)

The final working payload only prints the **first password** in the column due to the `LIMIT 1` constraint. To retrieve the administrator's password, we cannot simply use `LIMIT 2` (as that would print two passwords).

We need the **`OFFSET`** function to skip a specified number of rows from the top.

### Strategy using LIMIT 1 and OFFSET N

1.  **Retrieve 1st Password:** (The current payload)
    ```sql
    ' AND CAST((SELECT password FROM users LIMIT 1) AS BOOL)--
    ```
2.  **Retrieve 2nd Password:** (Skipping 1 row)
    ```sql
    ' AND CAST((SELECT password FROM users LIMIT 1 OFFSET 1) AS BOOL)--
    ```
3.  **Retrieve 3rd Password:** (Skipping 2 rows)
    ```sql
    ' AND CAST((SELECT password FROM users LIMIT 1 OFFSET 2) AS BOOL)--
    ```
    * By increasing the `OFFSET` value, we can iterate through the list of passwords until we retrieve the administrator's password .
4.  **Solve:** Once the administrator's password is found in the error message, log in.

---

## 5. Key Learnings/Concepts

* **Visible Error-Based SQLi:** Exploiting visible database error messages as an output channel for data exfiltration.
* **Casting for Error:** Forcing a data type conversion using `CAST(string AS non-compatible_type)` to trigger an explicit error that includes the original string.
* **Data Type Logic:** Understanding the database constraints, such as the `AND` clause requiring a **Boolean** type, which necessitated the switch from `AS INT` to `AS BOOL`.
* **Query Optimization:** Using `LIMIT` to shorten the query and satisfy injection restrictions.
* **Data Iteration (`LIMIT` & `OFFSET`):** Using `LIMIT 1 OFFSET N` is the standard, clean method to iterate through query results one row at a time, overcoming output limitations.

---

## 6. Mitigation

1.  **Parameterized Queries (Prepared Statements):** The primary defense. Prevents the injected SQL from being executed as code, making it impossible to inject the `AND CAST(...)` structure.
2.  **Input Validation:** Strictly validate and sanitize all input (especially cookies) to reject special characters like `'`, `--`, and SQL keywords.
3.  **Secure Error Handling:** **Never display raw database error messages** to users. All errors should be logged securely on the server side, and users should only see generic, non-informative error pages. This eliminates the oracle for Error-Based SQLi.
4.  **Least Privilege:** Configure the application's database account to have only the minimum necessary permissions, preventing it from executing complex functions like `CAST` or querying sensitive tables like `users`.
