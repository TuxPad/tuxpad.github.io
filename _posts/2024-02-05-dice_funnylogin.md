---
layout: post
title: DiceCTF 2024 - Funnylogin
date: 2024-02-05
categories: [CTF,Web]
tags: [CTF,SQLi,JS,Writeup]
toc: true
published: true
---

# Intro
Funnylogin is, as the name implies, a login bypass challenge.
The application, which is just a login page, is written in JavaScript, and the back end source code was provided for review.

The programming I've done has been mainly in Java and Python, with a touch of Go. I've had a look at JS, but never really used it to actually make any content. So a JS challenge is a good opportunity for me to learn something new.

# Reviewing the code
The first part sets up a SQLite database and imports the flag.
```javascript
const db = require('better-sqlite3')('db.sqlite3');
db.exec(`DROP TABLE IF EXISTS users;`);
db.exec(`CREATE TABLE users(
    id INTEGER PRIMARY KEY,
    username TEXT,
    password TEXT
);`);

const FLAG = process.env.FLAG || "dice{test_flag}";
```

Now, here's when things do get funny. The application creates an array of 100k users with random names and passwords and inserts them into the database. It then assigns an empty object to the constant isAdmin, randomly selects one of the users and inserts that user into the object, along with the boolean value true.

So that means the admin is selected randomly at runtime. An SQLi alone is not going to cut it, because the database itself has no idea which of the users is admin.

```javascript
const users = [...Array(100_000)].map(() => ({ user: `user-${crypto.randomUUID()}`, pass: crypto.randomBytes(8).toString("hex") }));
db.exec(`INSERT INTO users (id, username, password) VALUES ${users.map((u,i) => `(${i}, '${u.user}', '${u.pass}')`).join(", ")}`);

const isAdmin = {};
const newAdmin = users[Math.floor(Math.random() * users.length)];
isAdmin[newAdmin.user] = true;
```

Then we get to the actual login. Here we can see the query sent to the database.
The query itself returns the id value from the users table if the correct username and password is provided, and is then used by the application to check against the object referenced by isAdmin.

While the query itself is vulnerable to injection, as the user input is passed into it without filtering or validation, we still need to trick the application to let us bypass the login.

We thus need to fulfill the following conditions:
1. We need to provide credentials that return an id.
2. That id needs to be in the range of the users array.
3. The user provided need to evaluate to true for isAdmin.

```javascript
app.post("/api/login", (req, res) => {
    const { user, pass } = req.body;

    const query = `SELECT id FROM users WHERE username = '${user}' AND password = '${pass}';`;
    try {
        const id = db.prepare(query).get()?.id;
        if (!id) {
            return res.redirect("/?message=Incorrect username or password");
        }

        if (users[id] && isAdmin[user]) {
            return res.redirect("/?flag=" + encodeURIComponent(FLAG));
        }
        return res.redirect("/?message=This system is currently only available to admins...");
    }
    catch {
        return res.redirect("/?message=Nice try...");
    }
});
```

# Exploit
This is where I learned about <a href="https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Objects/Object_prototypes" target="_blank">object prototype chains</a> in JS. Having worked with classes in Java, this was a bit different, but it seems that non-null objects in JS contains a link to a prototype object, which in turn has a prototype, until reaching a null-object as the final link.

As such, all JS objects, including literals created using {} syntax inherits methods and constructors from Object.prototype, which includes toString(), hasOwnProperty(),valueOf(), `__proto__`, etc.

This means that passing any of these inherited properties as the username will do a boolean check against isAdmin, and since these properties exists, the check will return true.

Now all we have to do is make the query return an id that is within the range of the user array. In SQL, the first id starts at 1, but in JS the first index of an array is 0. This means that any value between 1 and 99,999 will be present in both the database and the array.

Our payload then will be:
username: `toString`
password: `'or id=1;--`

This bypasses the login and we're rewarded with the flag `dice{i_l0ve_java5cript!}`
