Your intent is **incremental systems-design practice**: build a small Python repository that progressively teaches **module boundaries → packaging → dependency stability → API stability → build reproducibility → CI stability**, without starting from Omnigent's full complexity.

# Incremental Project: `stable-modules`

## Goal

Build a small Python monorepo whose modules are deliberately designed to remain stable as the repository grows.

The final system should demonstrate this dependency structure:

$$
\text{Source Modules}
\rightarrow
\text{Public APIs}
\rightarrow
\text{Package Build}
\rightarrow
\text{Locked Environment}
\rightarrow
\text{Tests}
\rightarrow
\text{CI}
\rightarrow
\text{Release}
$$

You should **not** implement all of this at once. Each exercise introduces one stability mechanism.

---

# Phase 0 — Repository skeleton

### Exercise 0.1 — Create the package

Create:

```text
stable-modules/
├── pyproject.toml
├── src/
│   └── stable_modules/
│       ├── __init__.py
│       └── math.py
└── tests/
    └── test_math.py
```

Implement:

```python
# math.py
def add(x: int, y: int) -> int:
    ...
```

Test:

```python
def test_add():
    assert add(2, 3) == 5
```

### Invariant

The package has one import boundary:

$$
\texttt{tests}
\rightarrow
\texttt{stable\_modules}
$$

and never:

$$
\texttt{tests}
\rightarrow
\texttt{src/stable\_modules/math.py}
$$

The test consumes the **package interface**, not its filesystem location.

---

# Phase 1 — Internal vs public modules

### Exercise 1.1

Split the implementation:

```text
stable_modules/
├── __init__.py
├── arithmetic.py
└── _internal.py
```

Make:

```python
stable_modules.add
```

the public API.

The internal implementation should be inaccessible as part of the intended public contract.

### Exercise 1.2

Create:

```python
from stable_modules import add
```

as the supported usage.

Then deliberately change the implementation location:

```text
_internal.py
arithmetic.py
```

without changing:

```python
from stable_modules import add
```

### Question

What changed?

Formally, distinguish:

$$
I = \text{implementation representation}
$$

from:

$$
A = \text{public API}
$$

Your stability requirement is:

$$
I_1 \neq I_2
\quad\not\Rightarrow\quad
A_1 \neq A_2
$$

---

# Phase 2 — Explicit module boundaries

Add:

```text
stable_modules/
├── __init__.py
├── arithmetic.py
├── strings.py
└── errors.py
```

Define:

```python
class ArithmeticError(Exception):
    ...
```

and expose only:

```python
add
subtract
multiply
```

through the package.

### Exercise 2.1

Create an API surface:

```python
stable_modules.__all__
```

### Exercise 2.2

Try importing internal implementation symbols.

Decide which imports are:

* public
* private
* implementation detail

### Exercise 2.3

Write tests that test **only public behavior**.

Don't test:

```python
stable_modules.arithmetic._helper
```

unless `_helper` itself is part of the contract.

### Stability concept

You're learning:

> **A module boundary is a contract boundary.**

---

# Phase 3 — Dependency direction

Add:

```text
stable_modules/
├── core/
│   ├── __init__.py
│   ├── arithmetic.py
│   └── errors.py
│
├── formatting/
│   ├── __init__.py
│   └── numbers.py
│
└── app/
    ├── __init__.py
    └── service.py
```

Require:

$$
\text{app}
\rightarrow
\text{core}
$$

and:

$$
\text{formatting}
\rightarrow
\text{core}
$$

but prohibit:

$$
\text{core}
\rightarrow
\text{app}
$$

### Exercise 3.1

Introduce an accidental circular dependency.

Observe the failure.

### Exercise 3.2

Refactor until the dependency graph becomes acyclic.

You should be able to represent the package dependency relation as:

$$
G=(V,E)
$$

where:

$$
V=\{\text{core},\text{formatting},\text{app}\}
$$

and:

$$
E \subseteq V\times V
$$

with no directed cycle.

---

# Phase 4 — Compositional root

Now introduce:

```text
stable_modules/
├── core/
├── formatting/
├── storage/
└── app/
```

Create:

```python
app/bootstrap.py
```

The bootstrap module constructs the application.

For example:

```python
def build_app():
    storage = ...
    service = ...
    return ...
```

### Rule

Infrastructure gets assembled at the **composition root**.

Modules should not independently discover global infrastructure.

Bad:

```python
service = Service()
```

where `Service` internally creates a database.

Prefer:

```python
storage = SQLiteStorage(...)
service = Service(storage)
```

### Question

Identify the difference between:

$$
\text{construction}
$$

and:

$$
\text{behavior}
$$

The composition root owns construction.

The modules own behavior.

---

# Phase 5 — Storage boundary

Add:

```text
stable_modules/
├── domain/
│   └── user.py
├── storage/
│   ├── protocol.py
│   └── sqlite.py
└── app/
    └── service.py
```

Define a storage abstraction:

```python
class UserStore(Protocol):
    def get_user(self, user_id: int) -> User | None:
        ...
```

Implement:

```python
SQLiteUserStore
```

### Exercise 5.1

Make the application depend on:

```text
UserStore
```

rather than:

```text
SQLiteUserStore
```

### Exercise 5.2

Create:

```python
InMemoryUserStore
```

for tests.

Now:

$$
\text{Application}
\rightarrow
\text{UserStore}
\leftarrow
\begin{cases}
\text{SQLiteUserStore}\\
\text{InMemoryUserStore}
\end{cases}
$$

This is your first serious **stability boundary**.

---

# Phase 6 — Package metadata

Convert the repository into a real PEP 621 package.

Define:

```toml
[project]
name = "stable-modules"
version = "0.1.0"
requires-python = ">=3.12"
```

Add build configuration.

### Exercises

1. Build a wheel.
2. Build an sdist.
3. Install the wheel into a clean environment.
4. Import the package.
5. Run its tests against the installed package.

The important transition is:

$$
\text{source tree works}
\not\Rightarrow
\text{package artifact works}
$$

You are now testing the **distribution boundary**.

---

# Phase 7 — Versioned public API

Create:

```text
stable_modules/
├── __init__.py
└── version.py
```

Expose:

```python
stable_modules.__version__
```

### Exercise

Change:

```text
0.1.0 → 0.2.0
```

without breaking:

```python
from stable_modules import add
```

Then create a compatibility-sensitive change:

```python
add(x, y)
```

→

```python
add(x, y, *, checked=False)
```

Determine whether this is:

* backward compatible
* source compatible
* behavior compatible
* breaking

Write tests expressing your conclusion.

---

# Phase 8 — Dependency locking

Introduce one third-party dependency.

Then create a lockfile.

Your experiment is:

```text
pyproject.toml
        ↓
dependency resolver
        ↓
lockfile
        ↓
environment
```

### Exercise 8.1

Install from the lockfile.

### Exercise 8.2

Modify the declared dependency range.

Observe the difference between:

$$
\text{declared dependency constraint}
$$

and:

$$
\text{resolved dependency version}
$$

### Exercise 8.3

Delete the environment and reconstruct it exclusively from the lockfile.

### Stability invariant

Two developers starting from the same repository should obtain equivalent dependency graphs:

$$
L_1 = L_2
\Rightarrow
D_1 \approx D_2
$$

where \(L\) is the lock state and \(D\) is the resolved dependency graph.

---

# Phase 9 — Contract tests

Create:

```text
tests/
├── unit/
├── contract/
└── integration/
```

Your storage protocol now gets a contract test suite.

Conceptually:

$$
\text{Contract}
\rightarrow
\begin{cases}
\text{InMemory implementation}\\
\text{SQLite implementation}
\end{cases}
$$

Both implementations must satisfy the same behavioral predicates.

For example:

```python
def test_create_then_get(store):
    ...
```

Run it against both stores.

### Key idea

You aren't testing implementations separately first.

You're testing:

$$
\forall s \in \text{Implementations},
\quad
s \models C
$$

where \(C\) is the storage contract.

---

# Phase 10 — Build artifact freshness

Introduce generated code.

Create:

```text
schemas/
    user.json

scripts/
    generate_user.py

src/
    stable_modules/
        generated/
            user.py
```

The script transforms:

$$
S \xrightarrow{G} A
$$

where:

* \(S\) = schema
* \(G\) = generator
* \(A\) = generated artifact

### Exercise

Make CI fail if:

```text
generate(schema) != committed generated artifact
```

This teaches the same fundamental stability principle as generated protobuf bindings:

> **The checked-in artifact must correspond to the canonical source.**

---

# Phase 11 — Static enforcement

Add:

* formatter
* linter
* type checker
* test runner

Then create:

```text
scripts/check.py
```

which executes the repository's validation pipeline.

Conceptually:

$$
\text{Source}
\rightarrow
\begin{cases}
\text{format}\\
\text{lint}\\
\text{type check}\\
\text{tests}\\
\text{artifact check}
\end{cases}
$$

The important exercise is to deliberately introduce one failure of each type and identify **which layer owns detection**.

---

# Phase 12 — CI matrix

Create CI jobs for:

```text
unit
contract
integration
package
```

Then add Python versions.

Your matrix becomes approximately:

$$
\text{Tests}
\times
\text{Python Versions}
$$

For example:

```text
unit       × 3.12
unit       × 3.13
contract   × 3.12
contract   × 3.13
integration× 3.12
integration× 3.13
package    × 3.12
package    × 3.13
```

### Exercise

Make each job independently runnable.

Then ask:

> If `integration` fails, should `unit` have to rerun?

This introduces **failure-domain isolation**.

---

# Phase 13 — Compatibility shim

Now deliberately create an API migration.

Old:

```python
store.save(user)
```

New:

```python
store.put(user)
```

Create:

```text
_legacy_shim.py
```

that translates:

$$
f_{\text{old}} \rightarrow f_{\text{new}}
$$

Keep the shim for one release.

Add tests specifically for:

```text
legacy API → shim → new implementation
```

Then remove it.

This teaches:

* compatibility layers
* migration boundaries
* deprecation
* legacy protocol adaptation

---

# Phase 14 — Release pipeline

Create:

```text
scripts/
├── build.py
├── check_release.py
└── update_version.py
```

The release invariant becomes:

$$
\text{version}_{source}
=
\text{version}_{package}
=
\text{version}_{artifact}
$$

Build:

```text
dist/
├── stable_modules-X.Y.Z.tar.gz
└── stable_modules-X.Y.Z-py3-none-any.whl
```

Then test installation **from the wheel**, not the source tree.

---

# Phase 15 — Final architecture challenge

By the end, your repository should approximately have:

```text
stable-modules/
├── pyproject.toml
├── lockfile
├── README.md
│
├── src/
│   └── stable_modules/
│       ├── __init__.py
│       ├── version.py
│       │
│       ├── domain/
│       │   └── user.py
│       │
│       ├── core/
│       │   └── service.py
│       │
│       ├── storage/
│       │   ├── protocol.py
│       │   └── sqlite.py
│       │
│       ├── formatting/
│       │   └── numbers.py
│       │
│       └── _legacy_shim.py
│
├── schemas/
│   └── user.json
│
├── scripts/
│   ├── generate_user.py
│   ├── check.py
│   └── update_version.py
│
├── tests/
│   ├── unit/
│   ├── contract/
│   ├── integration/
│   └── package/
│
└── .github/
    └── workflows/
        └── ci.yml
```

## What each phase is actually teaching

| Phase | Engineering concept | Stability mechanism               |
| ----- | ------------------- | --------------------------------- |
| 0     | Package boundary    | Import contract                   |
| 1     | Public/private API  | Encapsulation                     |
| 2     | Module boundaries   | Controlled surface                |
| 3     | Dependency graph    | Acyclic architecture              |
| 4     | Composition root    | Construction isolation            |
| 5     | Ports/adapters      | Implementation substitution       |
| 6     | PEP 517/621         | Distribution boundary             |
| 7     | Versioning          | API evolution                     |
| 8     | Lockfiles           | Reproducible dependencies         |
| 9     | Contract tests      | Implementation interchangeability |
| 10    | Code generation     | Artifact consistency              |
| 11    | Static checks       | Pre-merge enforcement             |
| 12    | CI matrix           | Failure isolation                 |
| 13    | Legacy shim         | Backward compatibility            |
| 14    | Release             | Artifact reproducibility          |
| 15    | Integration         | System-level stability            |

The deeper model you're learning is:

$$
\boxed{
\text{Stable Module}
=
(\text{Boundary},\text{Contract},\text{Implementation},\text{Verification},\text{Evolution Rule})
}
$$

and the repository progressively adds constraints around each component rather than trying to make the entire system "stable" in one step.

A particularly useful capstone exercise is to **break one invariant intentionally at every phase, then add exactly one mechanism that prevents that class of breakage**. That makes the project teach *why* Omnigent has all those seemingly redundant layers rather than just copying its tooling.
