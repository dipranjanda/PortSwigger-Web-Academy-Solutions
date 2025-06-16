# 🧪 Lab 7: Determining the Number of Columns Returned by the Query (SQL Injection) Solution

**PortSwigger Lab Link:** [You'll insert the direct link to the PortSwigger Lab here when you upload it to GitHub]

**Table of Contents:**
* 1. Lab Description
* 2. Vulnerability
* 3. Exploitation Steps
* 4. Key Learnings/Concepts
* 5. Mitigation

---

## 1. Lab Description

This lab is a foundational exercise in **SQL Injection**, specifically focusing on the initial reconnaissance step of determining the **number of columns** returned by the application's backend SQL query. This information is crucial for successfully executing `UNION SELECT` attacks, as the number of selected columns in the injected query must match the number of columns in the original query. The goal is to achieve a successful `HTTP/200 OK` response using `UNION SELECT` with the correct number of `NULL` placeholders.

---

## 2. Vulnerability

The vulnerability here is a standard **SQL Injection**, where the application dynamically constructs SQL queries using user-supplied input without proper sanitization. This allows an attacker to inject SQL keywords and operators, such as `UNION SELECT`, to manipulate the original query.

While this specific lab doesn't extract sensitive data, it demonstrates the critical first step in many `UNION-based` SQL Injection attacks. By accurately determining the column count, an attacker can then proceed to inject further `UNION SELECT` statements to enumerate database schema, extract sensitive data, or even perform arbitrary commands in some cases. The ability to control the structure of the SQL query allows bypassing the intended application logic.

---

## 3. Exploitation Steps

### 🔹 Step 1: Intercept Request with Burp Suite

1.  Open the PortSwigger lab in your web browser.
2.  Ensure your browser is configured to proxy traffic through **Burp Suite**.
3.  Navigate to a page that likely uses a database query based on user input, such as a product filter or search page. For example, a URL might look like `https://example.com/filter?category=Lifestyle`.
4.  In Burp Suite's "Proxy" tab, intercept this request and send it to **Repeater** for detailed analysis and modification.

### 🔹 Step 2: Use `UNION SELECT` with `NULL` Placeholders to Determine Column Count

The rule of `UNION SELECT` is that the number of columns in the `SELECT` statements must match. If they don't, the database will return an error (e.g., "500 Internal Server Error" or "syntax error"). We can leverage this behavior to find the correct column count by progressively adding `NULL` placeholders until we get a successful HTTP 200 OK response.

1.  In Burp Repeater, locate the vulnerable parameter (e.g., `category`).
2.  Start injecting `UNION SELECT` payloads, gradually increasing the number of `NULL` placeholders. Remember to append `--` to comment out the rest of the original query.
    * **Attempt 1 (one column):**
        ```sql
        ' UNION SELECT NULL--
        ```
        * Send the request. If the original query returns more than one column, you will likely receive an `HTTP/500 Internal Server Error` or a similar database error.

    * **Attempt 2 (two columns):**
        ```sql
        ' UNION SELECT NULL, NULL--
        ```
        * Send the request. Continue to observe the response code and content.

    * **Attempt 3 (three columns):**
        ```sql
        ' UNION SELECT NULL, NULL, NULL--
        ```
        * Send the request.

3.  **Continue this process until you receive an `HTTP/200 OK` response.** This indicates that you have successfully matched the number of columns in the backend query.

* 📌 **In this lab, after trying different numbers, we found that `' UNION SELECT NULL, NULL, NULL--` resulted in an `HTTP/200 OK` response, confirming that the backend query fetches three columns.**

### 🔹 Step 3: URL Encode Payload (If Necessary)

As discussed in previous labs, if you are injecting directly into a URL parameter, certain characters in your payload will need to be URL encoded. Burp Suite can automate this:

1.  In Burp Repeater, right-click on your crafted payload within the request.
2.  Select "Convert selection" -> "URL" -> "URL-encode key characters" or "URL-encode all characters."

* **Example Encoded Payload (for illustration):**
    `%27%20UNION%20SELECT%20NULL%2C%20NULL%2C%20NULL--`

## 4. Key Learnings/Concepts

* **Importance of Column Count:** This lab highlights that knowing the exact number of columns in the original query is a fundamental prerequisite for most `UNION SELECT` SQL injection attacks.
* **`UNION SELECT NULL` Technique:** Using `NULL` placeholders is a reliable and database-agnostic way to determine the column count without worrying about data types.
* **Error-Based Detection:** Exploiting error messages (or the lack thereof) to infer information about the backend database structure. A transition from an error response to an `HTTP/200 OK` response is a clear indicator of success.
* **SQL Comments (`--`):** Essential for nullifying the remainder of the original query, ensuring that only your injected `UNION SELECT` statement is executed.
* **Burp Suite Repeater:** An indispensable tool for performing iterative testing, allowing quick modification of payloads and immediate observation of server responses.
* **HTTP Status Codes:** Understanding the significance of `HTTP/500` (Internal Server Error) indicating a syntax or database error, versus `HTTP/200 OK` indicating a successful query execution.

---

## 5. Mitigation

Preventing this type of SQL Injection involves the same robust security measures as other SQLi vulnerabilities:

* **Parameterized Queries (Prepared Statements):** This is the primary and most effective defense. By separating SQL code from user-supplied data, the database engine treats input as data values, not as executable commands, thus preventing injection.
* **Strong Input Validation:** While not a complete defense on its own, validating all user input against expected data types and formats can significantly reduce the attack surface.
* **Least Privilege Principle:** Ensure that database users and the application only have the minimal necessary permissions. This can limit what an attacker can achieve even if an injection is successful.
* **Secure Error Handling:** Avoid displaying detailed database error messages to users. Such errors can inadvertently provide attackers with valuable information about the backend database structure. Implement generic error messages and log detailed errors securely on the server side.
* **Web Application Firewalls (WAFs):** A WAF can detect and block known SQL injection patterns, adding an extra layer of protection, but should not be the sole defense.
