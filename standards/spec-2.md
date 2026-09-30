Yes. If your goal is to learn **Python standards as architecture**, I’d add a third exercise track around **application composition, API boundaries, and persistence**.

The key distinction is:

$$
\text{Entry Point}
\rightarrow
\text{Composition Root}
\rightarrow
\text{Application/API}
\rightarrow
\text{Domain}
\rightarrow
\text{Persistence}
$$

The standards/tools you want to practice are roughly:

* **PEP 621 / 517** — package/build metadata
* **PEP 8 / 257** — code/documentation conventions
* **typing + Protocol** — structural interfaces
* **dataclasses** — simple domain data
* **context managers** — resource lifetime
* **DB-API 2.0 (PEP 249)** — database interface
* **ASGI / WSGI** — application server interface
* **dependency injection by composition** — application architecture rather than a particular PEP
* **entry points** — package → executable/application boundary

## Track A — Database composition

### Exercise 1 — Raw DB boundary

Build:

```text
app/
    domain/
        user.py
    storage/
        database.py
    main.py
```

Start with SQLite.

Implement:

```python
def create_user(conn, user: User) -> None:
    ...

def find_user(conn, user_id: int) -> User | None:
    ...
```

The important constraint:

> `domain/user.py` must know nothing about SQLite.

You should be able to model:

$$
User \in Domain
$$

while:

$$
SQLiteConnection \in Infrastructure
$$

---

### Exercise 2 — DB-API 2.0

Use Python's standard DB interface:

```python
import sqlite3
```

Practice the DB-API concepts:

```python
conn
cursor
execute()
fetchone()
fetchall()
commit()
rollback()
```

Then identify the abstract structure:

$$
Connection
\rightarrow
Cursor
\rightarrow
SQL\ Operation
\rightarrow
Result
$$

Your exercise is to determine which pieces belong to the **database protocol** and which belong to SQLite specifically.

---

### Exercise 3 — Repository Protocol

Define:

```python
from typing import Protocol

class UserRepository(Protocol):
    def save(self, user: User) -> None: ...
    def find(self, user_id: int) -> User | None: ...
```

Then implement:

```python
class SQLiteUserRepository:
    ...
```

Now the dependency direction becomes:

$$
Application \rightarrow UserRepository
$$

rather than:

$$
Application \rightarrow SQLiteUserRepository
$$

This is one of the most useful exercises for understanding `Protocol`.

---

## Track B — Composition Root

### Exercise 4 — Remove construction from business logic

Initially write:

```python
def register_user(name: str):
    db = sqlite3.connect("app.db")
    repo = SQLiteUserRepository(db)

    ...
```

Then refactor.

The application function should instead receive:

```python
def register_user(
    repo: UserRepository,
    name: str,
) -> User:
    ...
```

Now you have:

$$
f : Repository \times Input \rightarrow Result
$$

rather than:

$$
f : Input \rightarrow Result
$$

where `f` secretly constructs infrastructure.

---

### Exercise 5 — Actual Composition Root

Create:

```text
src/app/
    domain/
    application/
    infrastructure/
    composition.py
    cli.py
```

Make:

```python
def compose_application() -> Application:
    conn = sqlite3.connect("app.db")
    repo = SQLiteUserRepository(conn)

    return Application(repo)
```

This function becomes your **composition root**.

Its job is essentially:

$$
\text{Concrete Components}
\rightarrow
\text{Dependency Graph}
\rightarrow
\text{Application}
$$

Nothing below it should decide:

* which database to use
* which repository implementation to use
* which HTTP client to use
* which configuration source to use

---

# Track C — API boundary

Now add an HTTP API.

### Exercise 6 — API as an adapter

Don't put business logic inside the route.

Instead:

```python
def create_user_endpoint(request):
    command = ...
    result = application.register_user(command)
    return ...
```

The API becomes:

$$
HTTP
\rightarrow
Adapter
\rightarrow
Application
\rightarrow
Domain
$$

The API shouldn't know SQL.

The repository shouldn't know HTTP.

The domain shouldn't know either.

---

### Exercise 7 — API dependency composition

Make your API application object explicit:

```python
class Application:
    def __init__(
        self,
        users: UserRepository,
    ):
        self.users = users
```

Then the composition root creates:

```python
repo = SQLiteUserRepository(...)
app = Application(repo)
api = create_api(app)
```

Now you have a clean construction sequence:

$$
Database
\rightarrow
Repository
\rightarrow
Application
\rightarrow
API
$$

---

# Track D — Entry points

### Exercise 8 — CLI entry point

Expose:

```toml
[project.scripts]
myapp = "app.cli:main"
```

Then:

```bash
myapp create-user Alice
```

Your CLI should **not** construct individual repositories itself.

Instead:

```python
def main() -> int:
    app = compose_application()
    ...
```

So:

$$
OS
\rightarrow
CLI\ Entry\ Point
\rightarrow
Composition\ Root
\rightarrow
Application
$$

---

### Exercise 9 — API entry point

If you use an ASGI framework/server, make the entry point expose an application object:

```python
app = create_api(compose_application())
```

Conceptually:

$$
ASGI\ Server
\rightarrow
app
\rightarrow
Application
\rightarrow
Repository
\rightarrow
Database
$$

Now compare this with the CLI:

$$
CLI
\rightarrow
main()
\rightarrow
Application
$$

Both are **adapters into the same application core**.

---

# Track E — Resource lifetime

This is particularly important for your functional architecture preferences.

### Exercise 10 — Database lifetime

Start with:

```python
conn = sqlite3.connect(...)
```

Determine:

* who creates it?
* who owns it?
* who closes it?
* how long does it live?

Then experiment with:

```python
with sqlite3.connect(...) as conn:
    ...
```

Model the lifetime as:

$$
Created
\rightarrow
Active
\rightarrow
Closed
$$

Then ask:

> Should `Application` own the connection, or should the composition root own the connection?

Don't assume the answer—make the lifetime constraint explicit.

---

# Track F — Testing the composition root

### Exercise 11 — In-memory replacement

Create:

```python
class InMemoryUserRepository:
    ...
```

Then your production graph is:

$$
Application
\rightarrow
SQLiteUserRepository
\rightarrow
SQLite
$$

while your test graph is:

$$
Application
\rightarrow
InMemoryUserRepository
$$

The **application code does not change**.

Only:

$$
Composition : Configuration \rightarrow DependencyGraph
$$

changes.

That's a very useful way to understand dependency injection without introducing a DI framework.

---

# Final project

Build this:

```text
myapp/
├── pyproject.toml
├── src/
│   └── myapp/
│       ├── domain/
│       │   └── user.py
│       ├── application/
│       │   └── users.py
│       ├── infrastructure/
│       │   └── sqlite_users.py
│       ├── api/
│       │   └── http.py
│       ├── cli.py
│       └── composition.py
└── tests/
    ├── test_domain.py
    ├── test_application.py
    └── test_api.py
```

With the dependency structure:

$$
\boxed{
Entry\ Point
\rightarrow
Composition\ Root
\rightarrow
Application
\rightarrow
Ports
\rightarrow
Adapters
\rightarrow
Infrastructure
}
$$

and:

$$
\text{Infrastructure}
\not\rightarrow
\text{Entry Point}
$$

$$
\text{Domain}
\not\rightarrow
\text{Infrastructure}
$$

$$
\text{Application}
\not\rightarrow
\text{Concrete Infrastructure}
$$

### The standards/concepts I'd specifically study alongside it

| Area             | Python standard/concept            | Exercise purpose      |
| ---------------- | ---------------------------------- | --------------------- |
| Package metadata | PEP 621                            | Describe project      |
| Build            | PEP 517                            | Build artifact        |
| Style            | PEP 8                              | Code conventions      |
| Docs             | PEP 257                            | API documentation     |
| Interfaces       | `typing.Protocol`                  | Ports                 |
| Data             | `dataclasses`                      | Domain values         |
| DB               | **PEP 249 / DB-API 2.0**           | Database boundary     |
| Resources        | Context managers                   | Connection lifetime   |
| CLI              | `project.scripts`                  | Entry point           |
| HTTP             | **WSGI / ASGI**                    | API boundary          |
| Composition      | No special PEP                     | Dependency graph      |
| Testing          | `pytest` / `unittest`              | Alternate composition |
| Configuration    | `os.environ`, `configparser`, etc. | Runtime configuration |

The particularly valuable progression for you is:

$$
\text{PEP 621}
\rightarrow
\text{Entry Point}
\rightarrow
\text{Composition Root}
\rightarrow
\text{Protocol}
\rightarrow
\text{Repository}
\rightarrow
\text{DB-API}
\rightarrow
\text{Database}
$$

That gives you practice moving from **Python package semantics → executable boundary → dependency graph → abstract interface → concrete infrastructure**, which is much closer to the architecture work you've been doing than studying PEPs individually.
