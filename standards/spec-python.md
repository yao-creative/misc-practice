Your **intent: incremental implementation practice** — I’d structure these as two tracks that converge into one small package. The goal is to make each exercise introduce exactly one new abstraction before composing them.

## 5. PEP 20 — The Zen of Python

**Target:** practice turning Python design principles into concrete architectural decisions rather than memorizing the aphorisms.

### Level 1 — Readability as a constraint

**Exercise 1: Two equivalent implementations**

Implement:

```python
def group_by_first_letter(words: list[str]) -> dict[str, list[str]]:
    ...
```

Create two implementations:

* one deliberately compressed into clever Python
* one following PEP 20 principles

Requirements:

* no nested comprehensions
* meaningful names
* explicit intermediate values

**Question:** Which implementation has the smaller *semantic surface area*?

---

### Level 2 — Explicit vs implicit

**Exercise 2: Configuration lookup**

Implement:

```python
class Config:
    def get(self, key: str) -> str | None:
        ...
```

Support:

```python
config.get("database_url")
config.get("missing")
```

Then implement a second API:

```python
config["database_url"]
```

Compare:

* explicit absence
* implicit exception
* caller responsibility

Explain which API better communicates its contract.

---

### Level 3 — One obvious way

**Exercise 3: Package API**

Given:

```text
mypackage/
    parser.py
    loader.py
    models.py
    utils.py
```

Create:

```python
mypackage.parse(...)
mypackage.load(...)
```

Decide:

* what belongs in `__init__.py`
* what is public
* what is internal
* whether `utils.py` should exist

**Constraint:** Every public operation should have one obvious import path.

---

### Level 4 — Errors should never pass silently

**Exercise 4: Configuration parser**

Parse:

```text
PORT=8080
HOST=localhost
DEBUG=true
```

Handle:

```text
PORT=abc
UNKNOWN=value
DEBUG=maybe
```

Design:

```python
Result = ...
```

or ordinary Python exceptions.

Answer:

1. Which errors are programmer errors?
2. Which are user-input errors?
3. Which should be recoverable?
4. Which should terminate configuration loading?

---

### Level 5 — Flat is better than nested

**Exercise 5: Refactor a state machine**

Start with deliberately nested code:

```python
if request:
    if request.authenticated:
        if request.valid:
            if request.method == "POST":
                ...
```

Refactor it into guard clauses.

Then formalize the transformation as:

$$
f : X \rightarrow Y
$$

where each validation stage is a partial restriction of the input set.

---

### Level 6 — Namespaces are a great idea

**Exercise 6: Replace a utility module**

Start with:

```python
from utils import parse
from utils import validate
from utils import load
```

Refactor into:

```python
from config import parser
from config import validator
from config import loader
```

Then ask:

> What semantic information does the namespace encode that the function name alone does not?

---

### Level 7 — Refactoring exercise

**Exercise 7: Zen audit**

Take a small Python module you've already written.

For each applicable PEP 20 principle, record:

| Principle           | Current code | Problem | Refactoring |
| ------------------- | ------------ | ------- | ----------- |
| Explicit > implicit | ?            | ?       | ?           |
| Simple > complex    | ?            | ?       | ?           |
| Flat > nested       | ?            | ?       | ?           |
| Readability counts  | ?            | ?       | ?           |
| One obvious way     | ?            | ?       | ?           |

The important part is **not** satisfying every aphorism. Identify where two principles conflict.

---

# 4. PEP 621 + PEP 517 — Metadata & Packaging

**Target: packaging architecture.**

Here I'd make this much more implementation-heavy.

PEP 621 concerns the **declarative project metadata** in `pyproject.toml`; PEP 517 defines the **build-system interface** between a frontend and a backend.

Think of the architecture as:

$$
\text{Source Tree}
\rightarrow
\text{Build Backend}
\rightarrow
\text{Artifact}
$$

while:

$$
\text{pyproject.toml}
\rightarrow
\text{Project Metadata}
$$

---

## Level 1 — Minimal package

**Exercise 1: Create a package**

Create:

```text
hello_pkg/
├── pyproject.toml
├── src/
│   └── hello_pkg/
│       ├── __init__.py
│       └── greeting.py
└── tests/
    └── test_greeting.py
```

Implement:

```python
def hello(name: str) -> str:
    return f"Hello, {name}!"
```

---

## Level 2 — PEP 621 metadata

**Exercise 2: Write the project metadata**

Create:

```toml
[project]
name = "hello-pkg"
version = "0.1.0"
description = "..."
requires-python = ">=3.12"
dependencies = []
```

Then add:

```toml
[project.urls]
Repository = "..."
```

Questions:

1. What information belongs to `[project]`?
2. What information belongs to the build system?
3. Why shouldn't package metadata be scattered across arbitrary tool configuration?

---

## Level 3 — Dependencies

**Exercise 3: Add a runtime dependency**

Add one external dependency.

Represent:

$$
D = \{d_1,d_2,\ldots,d_n\}
$$

where each dependency is a package requirement.

Then distinguish:

$$
D_{\text{runtime}}
\neq
D_{\text{development}}
$$

Move testing dependencies into an optional dependency group.

For example, conceptually:

```toml
[project.optional-dependencies]
test = [...]
```

Then install the package with the test extras.

---

## Level 4 — Entry points

**Exercise 4: Build a CLI**

Implement:

```bash
hello-pkg Yi
```

which calls:

```python
def main() -> int:
    ...
```

Expose it through:

```toml
[project.scripts]
hello-pkg = "hello_pkg.cli:main"
```

Now answer:

> What changed?

You did **not** change the Python function's semantics.

You changed the mapping:

$$
\text{OS command}
\rightarrow
\text{Python callable}
$$

This is an important packaging abstraction.

---

## Level 5 — Build the wheel

**Exercise 5: Build an artifact**

Build the package into:

```text
dist/
    hello_pkg-0.1.0-py3-none-any.whl
    hello_pkg-0.1.0.tar.gz
```

Inspect the wheel.

Find:

```text
*.dist-info/
```

and identify:

```text
METADATA
WHEEL
RECORD
entry_points.txt
```

Question:

> Which information from `pyproject.toml` ended up inside the built artifact?

---

## Level 6 — PEP 517 mental model

**Exercise 6: Identify the actors**

Given:

```bash
python -m build
```

identify:

$$
\text{build frontend}
\rightarrow
\text{build backend}
\rightarrow
\text{artifact}
$$

Then answer:

* Who reads `pyproject.toml`?
* Who invokes the backend?
* Who creates the wheel?
* Who creates the sdist?
* Where does the backend come from?

Don't implement anything yet.

---

## Level 7 — Backend interface

**Exercise 7: Trace the build hooks**

Investigate the PEP 517 hooks:

```python
build_wheel(...)
build_sdist(...)
prepare_metadata_for_build_wheel(...)
```

Create a tiny fake backend:

```python
def build_wheel(...):
    ...

def build_sdist(...):
    ...

def prepare_metadata_for_build_wheel(...):
    ...
```

You don't need to produce a valid wheel yet.

Your goal is to understand:

$$
B : \text{BuildRequest} \rightarrow \text{Artifact}
$$

and that the frontend interacts with the backend through a **protocol**, rather than knowing how the backend performs the build.

---

## Level 8 — Backend swap

**Exercise 8: Change build backend**

Take the same source tree and switch build backends.

Your package's Python source should remain unchanged.

Only packaging configuration changes.

Verify:

```text
same source
      ↓
different backend
      ↓
build artifact
```

Then inspect whether the resulting metadata/artifacts differ.

This is the key PEP 517 lesson:

> **The build frontend should not need to understand the backend's implementation.**

---

## Level 9 — Metadata validation

**Exercise 9: Break the metadata intentionally**

Introduce errors:

```toml
version = "not-a-version"
```

or remove:

```toml
name = ...
```

or specify an invalid requirement.

Run the build.

Classify the failure:

$$
\text{source error}
\quad|\quad
\text{metadata error}
\quad|\quad
\text{backend error}
\quad|\quad
\text{frontend error}
$$

This is excellent practice for distinguishing **configuration contracts from implementation errors**.

---

# Final integration project

## Exercise 10 — Build `mini-tool`

Create a real package:

```text
mini-tool/
├── pyproject.toml
├── src/
│   └── mini_tool/
│       ├── __init__.py
│       ├── cli.py
│       ├── parser.py
│       └── errors.py
├── tests/
│   ├── test_parser.py
│   └── test_cli.py
└── README.md
```

Requirements:

### Python API

```python
from mini_tool import parse
```

### CLI

```bash
mini-tool input.txt
```

### Metadata

Declare:

* name
* version
* description
* Python requirement
* runtime dependencies
* optional test dependencies
* project URLs
* CLI entry point

### Build

Produce:

```text
sdist
wheel
```

### Installation test

Create a fresh virtual environment and verify:

```bash
pip install dist/...
mini-tool input.txt
```

---

# Final conceptual exercise

After completing the project, write down the sets:

$$
S = \{\text{source files}\}
$$

$$
M = \{\text{project metadata}\}
$$

$$
B = \{\text{build-backend operations}\}
$$

$$
A = \{\text{build artifacts}\}
$$

and the mappings:

$$
S \times M
\xrightarrow{\text{backend}}
A
$$

and:

$$
\text{CLI invocation}
\xrightarrow{\text{entry-point mapping}}
\text{Python callable}
$$

Then explain the distinction between:

**PEP 621:**
“What is this Python project?”

**PEP 517:**
“How does a packaging frontend ask a backend to build it?”

**PEP 20:**
“How should we make the Python implementation understandable while doing all this?”

That gives you a nice progression from **code-level design → package-level metadata → build-system protocol**.
