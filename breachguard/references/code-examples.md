# Platform-Code-Beispiele fuer haeufige Findings

Bad/Good-Snippets fuer die haeufigsten Findings in den groessten Sprach-
Oekosystemen. Zur Verwendung im **Developer-Rolle**-Output (siehe `roles.md`).

## SQL Injection (Lens: `injection`, CWE-89)

### Python (psycopg2)

**Bad:**
```python
cur.execute("SELECT * FROM users WHERE id = " + user_id)
cur.execute(f"SELECT * FROM users WHERE email = '{email}'")
```

**Good:**
```python
cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
cur.execute("SELECT * FROM users WHERE email = %s", (email,))
```

### Node.js (pg)

**Bad:**
```js
client.query(`SELECT * FROM users WHERE id = ${req.params.id}`);
```

**Good:**
```js
client.query('SELECT * FROM users WHERE id = $1', [req.params.id]);
```

### Java (JDBC)

**Bad:**
```java
Statement s = conn.createStatement();
s.executeQuery("SELECT * FROM users WHERE id = " + userId);
```

**Good:**
```java
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
ps.setInt(1, userId);
ResultSet rs = ps.executeQuery();
```

### Go (database/sql)

**Bad:**
```go
db.Query("SELECT * FROM users WHERE id = " + userID)
```

**Good:**
```go
db.Query("SELECT * FROM users WHERE id = ?", userID)
// or for PostgreSQL:
db.Query("SELECT * FROM users WHERE id = $1", userID)
```

## XSS (Lens: `xss-csrf`, CWE-79)

### React

**Bad:**
```jsx
<div dangerouslySetInnerHTML={{ __html: userComment }} />
```

**Good:**
```jsx
<div>{userComment}</div>  {/* React escapes by default */}

// If HTML is truly needed, sanitize first:
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userComment) }} />
```

### Vue

**Bad:**
```html
<div v-html="userComment"></div>
```

**Good:**
```html
<div>{{ userComment }}</div>  <!-- Vue escapes interpolation -->

<!-- If HTML needed: sanitize first -->
<div v-html="DOMPurify.sanitize(userComment)"></div>
```

### Server-Side (Node/Express + EJS)

**Bad:**
```html
<p><%- userComment %></p>  <%# unescaped output %>
```

**Good:**
```html
<p><%= userComment %></p>  <%# escaped output %>
```

## Hardcoded Secrets (Lens: `secrets`, CWE-798)

### Python

**Bad:**
```python
API_KEY = "sk-abc123xyz"
DATABASE_URL = "postgres://user:password123@prod-db.company.com/app"
```

**Good:**
```python
import os
API_KEY = os.environ["API_KEY"]
DATABASE_URL = os.environ["DATABASE_URL"]

# Or with python-decouple / pydantic-settings for structured config:
from pydantic_settings import BaseSettings
class Settings(BaseSettings):
    api_key: str
    database_url: str
settings = Settings()
```

### Node.js

**Bad:**
```js
const stripe = require('stripe')('sk_live_abc123xyz');
```

**Good:**
```js
require('dotenv').config();
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
// .env file MUST be in .gitignore
```

### Java

**Bad:**
```java
private static final String JWT_SECRET = "mySecretKey123";
```

**Good:**
```java
// With Spring Boot:
@Value("${jwt.secret}")
private String jwtSecret;
// Config via env var SPRING_APPLICATION_JSON or secrets manager

// Plain Java:
private static final String JWT_SECRET = System.getenv("JWT_SECRET");
if (JWT_SECRET == null) throw new IllegalStateException("JWT_SECRET not set");
```

## Missing Input Validation (Lens: `input-sanitization`, CWE-20)

### Python (FastAPI)

**Bad:**
```python
@app.post("/user")
def create_user(data: dict):
    return db.create(name=data["name"], age=data["age"])
```

**Good:**
```python
from pydantic import BaseModel, constr, conint

class UserCreate(BaseModel):
    name: constr(min_length=1, max_length=100)
    age: conint(ge=0, le=150)

@app.post("/user")
def create_user(user: UserCreate):
    return db.create(name=user.name, age=user.age)
```

### Node.js (Express + Zod)

**Bad:**
```js
app.post('/user', (req, res) => {
  db.create({name: req.body.name, age: req.body.age});
});
```

**Good:**
```js
import { z } from 'zod';
const UserSchema = z.object({
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150),
});

app.post('/user', (req, res) => {
  const user = UserSchema.parse(req.body);  // throws on invalid input
  db.create(user);
});
```

## Path Traversal (Lens: `injection` oder `input-sanitization`, CWE-22)

### Python

**Bad:**
```python
@app.get("/download/{filename}")
def download(filename: str):
    return FileResponse(f"/var/files/{filename}")
# Attacker: /download/../../etc/passwd
```

**Good:**
```python
from pathlib import Path
BASE = Path("/var/files").resolve()

@app.get("/download/{filename}")
def download(filename: str):
    target = (BASE / filename).resolve()
    if not target.is_relative_to(BASE):
        raise HTTPException(403, "Invalid path")
    return FileResponse(target)
```

### Node.js

**Bad:**
```js
app.get('/download/:filename', (req, res) => {
  res.sendFile(`/var/files/${req.params.filename}`);
});
```

**Good:**
```js
const path = require('path');
const BASE = path.resolve('/var/files');

app.get('/download/:filename', (req, res) => {
  const target = path.resolve(BASE, req.params.filename);
  if (!target.startsWith(BASE + path.sep)) {
    return res.status(403).send('Invalid path');
  }
  res.sendFile(target);
});
```

## Command Injection (Lens: `injection`, CWE-78)

### Python

**Bad:**
```python
import os
os.system(f"convert {user_file} output.png")
subprocess.run(f"grep {query} /var/log/app.log", shell=True)
```

**Good:**
```python
import subprocess
subprocess.run(["convert", user_file, "output.png"], check=True)
# No shell interpretation, arguments passed directly
subprocess.run(["grep", query, "/var/log/app.log"], check=True)
```

### Node.js

**Bad:**
```js
const { exec } = require('child_process');
exec(`convert ${userFile} output.png`);
```

**Good:**
```js
const { execFile } = require('child_process');
execFile('convert', [userFile, 'output.png'], (err, stdout) => {...});
// or spawn() for streaming
```

## Verwendung

Diese Snippets sind **Referenz fuer Developer-Rolle** (siehe `roles.md`).
Wenn der User Developer-Rolle triggert und eines dieser Findings vorliegt:
den passenden Before/After-Snippet in den Fix-Block einbauen, angepasst an
die konkrete Code-Stelle des Users.
