# Code Manifesto

This document exists so every contributor writes code the rest of the team can read, trust, and change without fear. It is not a style guide for semicolons — it is a shared definition of "good code" for this project. Examples below use Python since that's our stack, but the principles are language-agnostic.

If a pull request violates something here, that's a valid reason to request changes.

## 1. Clean Code, in Plain Terms

Code is read far more often than it's written. Optimize for the next person (often future-you) who has to understand it under time pressure.

- **One purpose per unit.** A function does one thing, a class represents one concept, a module owns one responsibility.
- **Small is good.** If a function doesn't fit on one screen, it's probably doing too much. Extract and name the pieces.
- **Good code is small parts that work, composed.** A complex system isn't one clever function that does everything — it's many small, individually-correct functions/classes wired together. Each small part should be simple enough to be *obviously* correct on its own, and simple enough to verify that it is. If you can't tell whether a piece of code is correct just by looking at it and its tests, it's too big or too tangled — split it. (This is exactly what unit testing in §8 is for: it's the mechanism that proves each small part actually works before you trust the composition.)
- **Names carry meaning.** `compute_r_squared(residuals)` beats `calc(x)`. Don't abbreviate to save keystrokes; the IDE autocompletes for you.
- **No magic values.** `MAX_RETRIES = 3` instead of a bare `3` showing up in an `if`.
- **Fail loudly, not silently.** Don't swallow exceptions with a bare `except: pass`. If you can't handle it, let it propagate or raise something meaningful.
- **Consistent abstraction level.** Don't mix low-level index arithmetic with high-level domain logic in the same function.

```python
# Bad
def p(d):
    r = []
    for x in d:
        if x[2] > 0.5:
            r.append(x)
    return r

# Good
def filter_significant_results(records: list[Result]) -> list[Result]:
    return [r for r in records if r.p_value > SIGNIFICANCE_THRESHOLD]
```

## 2. DRY — Don't Repeat Yourself

If you copy-paste a block of logic and tweak two variable names, stop. That's a signal it should be a function, a shared class, or a configuration value.

- Repeated logic means every future bug fix or change has to be applied N times, and someone always forgets one.
- DRY applies to **knowledge**, not just text. Two functions that happen to look similar but express different domain rules are *not* a DRY violation — don't force a shared abstraction just because the code looks alike.
- Constants, magic thresholds, and file paths belong in one place (config/constants module), not copy-pasted across scripts.

```python
# Bad: same normalization logic duplicated in two places
def load_train(path):
    df = pd.read_csv(path)
    df["x"] = (df["x"] - df["x"].mean()) / df["x"].std()
    return df

def load_test(path):
    df = pd.read_csv(path)
    df["x"] = (df["x"] - df["x"].mean()) / df["x"].std()
    return df

# Good
def normalize(df: pd.DataFrame, column: str) -> pd.DataFrame:
    df[column] = (df[column] - df[column].mean()) / df[column].std()
    return df

def load_dataset(path: str) -> pd.DataFrame:
    return normalize(pd.read_csv(path), "x")
```

## 3. SOLID

SOLID is about designing classes/modules so change is localized and cheap. In a research/simulation codebase, this mostly matters at the boundaries: data loading, models, and analysis/plotting should not be tangled together.

### S — Single Responsibility Principle
A class/module should have exactly one reason to change.

```python
# Bad: one class loads data, runs the simulation, AND plots results
class Experiment:
    def load_data(self): ...
    def run_simulation(self): ...
    def plot_results(self): ...

# Good: split by responsibility
class DataLoader:
    def load(self, path: str) -> pd.DataFrame: ...

class Simulation:
    def run(self, data: pd.DataFrame) -> SimulationResult: ...

class ResultPlotter:
    def plot(self, result: SimulationResult) -> None: ...
```

### O — Open/Closed Principle
You should be able to extend behavior without modifying existing, tested code.

```python
# Bad: adding a new model means editing this function and risking the existing branches
def run_model(name, data):
    if name == "sir":
        return run_sir(data)
    elif name == "seir":
        return run_seir(data)
    # every new model = another elif here

# Good: new models plug in without touching existing ones
class EpidemicModel(Protocol):
    def run(self, data: pd.DataFrame) -> SimulationResult: ...

MODEL_REGISTRY: dict[str, EpidemicModel] = {
    "sir": SIRModel(),
    "seir": SEIRModel(),
}

def run_model(name: str, data: pd.DataFrame) -> SimulationResult:
    return MODEL_REGISTRY[name].run(data)
```

### L — Liskov Substitution Principle
A subclass must be usable anywhere its parent is expected, without surprising behavior.

```python
# Bad: subclass breaks the contract (raises instead of returning), callers can't trust the interface
class Model:
    def run(self, data) -> SimulationResult: ...

class BrokenModel(Model):
    def run(self, data) -> SimulationResult:
        raise NotImplementedError("not ready")  # silently breaks any code looping over models

# Good: every subclass genuinely honors the parent's contract
class SIRModel(Model):
    def run(self, data) -> SimulationResult:
        return SimulationResult(...)
```

### I — Interface Segregation Principle
Don't force a class to implement methods it doesn't need. Prefer small, focused interfaces over one bloated one.

```python
# Bad: forces every model to implement plotting even if it never plots
class Model(Protocol):
    def run(self, data): ...
    def plot(self, result): ...
    def export_to_csv(self, result): ...

# Good: split into focused protocols, implement only what applies
class Runnable(Protocol):
    def run(self, data): ...

class Plottable(Protocol):
    def plot(self, result): ...
```

### D — Dependency Inversion Principle
Depend on abstractions, not concrete implementations. High-level logic shouldn't reach into low-level details directly.

```python
# Bad: Simulation is hard-wired to one specific data source
class Simulation:
    def __init__(self):
        self.loader = CSVLoader("data/raw.csv")

# Good: the dependency is injected, so it's swappable and testable
class Simulation:
    def __init__(self, loader: DataLoader):
        self.loader = loader

Simulation(loader=CSVLoader("data/raw.csv"))
Simulation(loader=MockLoader())  # trivial to test
```

## 4. Self-Documenting Code

The code itself should explain *what* it does. Comments should explain *why*, when the "why" isn't obvious from reading it.

- If you need a comment to explain what a block does, first try renaming variables/functions or extracting a well-named helper instead.
- Type hints are documentation that the interpreter and linters can check for you. Use them on all public functions.
- Docstrings are for the *contract* of a function (inputs, outputs, side effects, invariants) — not a restatement of the code.

```python
# Bad: comment compensates for a bad name
# check if user can vote
def chk(u):
    return u.a >= 18

# Good: the code reads like the comment
def is_eligible_to_vote(user: User) -> bool:
    return user.age >= VOTING_AGE
```

## 5. Comments: Only When Necessary

Default to no comments. Treat every comment as a liability, not a nicety.

**Why comments are actively bad, not just neutral:** code changes constantly; comments don't change with it automatically. A comment is not executed, not type-checked, not tested — nothing forces it to stay in sync with the logic beneath it. In practice, whoever edits the code three months from now is fixing a bug, is in a hurry, and does not re-read (let alone update) every comment near their change. The result is a comment that now describes behavior the code no longer has — and a comment that's actively wrong is worse than no comment at all, because it's actively misleading, and it's often trusted *more* than the code by whoever reads it next. "Self-documenting code" (§4) doesn't rot this way: if you rename a variable or change a condition, the "documentation" changes with it because it *is* the code.

So the default answer to "should I add a comment?" is no — go make the code clearer instead (better names, an extracted function, a type hint). A comment is justified only when it captures something the code fundamentally *cannot* express on its own:

- **Why**, not what: "Using a manual loop here instead of vectorized ops because N < 20 and readability wins" — not "loop over items".
- Non-obvious trade-offs, workarounds for library bugs, links to the paper/issue behind a formula.
- `TODO:` / `FIXME:` with context, when something is intentionally incomplete.

If you do write one, treat it as part of the diff: any PR that changes code near a comment must also update or delete that comment in the same commit. A comment you didn't have to touch during a change is a comment that's already drifting.

```python
# Bad: restates the obvious, will rot the moment the code changes
# increment i by 1
i += 1

# Good: explains a non-obvious reason
# Newton-Raphson can overshoot near R0 ≈ 1; clamp to avoid divergence (see docs/final_project_css.pdf, sec. 3.2)
r0 = max(r0, MIN_R0)
```

## 6. Python-Specific Rules

Python gives you two ways to process data: explicit loops, or vectorized/columnar operations. Default to the latter.

- **Avoid manual `for` loops over data.** A Python-level loop over rows/elements is slow, harder to read at a glance, and usually hides an operation that a library already expresses in one line. If you're looping to filter, transform, aggregate, or accumulate over a collection of numbers/rows, that's a signal to reach for `numpy`/`pandas` instead.
- **Use NumPy for numerical/array work.** Vectorized array ops instead of element-by-element Python loops — faster, and reads as the math it represents.
- **Use pandas for tabular data.** Column-wise operations, `groupby`, `merge`, boolean masking — instead of iterating rows with `.iterrows()`/`for row in df`, which is both slow and an anti-pattern in pandas.
- **A manual loop is acceptable when:** the collection is tiny (see §5's example, N < 20) and a loop is genuinely more readable than the vectorized equivalent, or the logic is inherently sequential/stateful (e.g. an iterative simulation step that depends on the previous one) and can't be vectorized. If you take this exception, say why in a comment (see §5) — that's exactly the kind of "why" that justifies one.
- **Use Spark for big data.** Do we have access to a cluster?? If a dataset doesn't comfortably fit in memory / on one machine, or the pipeline needs to be distributed, use PySpark rather than trying to force pandas to scale (e.g. chunking a `.csv` by hand). Don't reach for Spark by default though — it has real overhead and is overkill for anything that fits comfortably in a pandas `DataFrame`. Match the tool to the actual data size. 

```python
# Bad: Python-level loop to sum/filter/transform
totals = []
for row in df.itertuples():
    if row.infected > 0:
        totals.append(row.infected * row.weight)
result = sum(totals)

# Good: vectorized, reads as the math it represents
result = (df.loc[df["infected"] > 0, "infected"] * df.loc[df["infected"] > 0, "weight"]).sum()
```

```python
# Bad: manual accumulation loop over an array
squared_errors = []
for i in range(len(predictions)):
    squared_errors.append((predictions[i] - actual[i]) ** 2)
mse = sum(squared_errors) / len(squared_errors)

# Good: NumPy vectorized
mse = np.mean((predictions - actual) ** 2)
```

## 7. OOP Pillars

Use classes when they model something with real state and behavior — not just to wrap a bag of functions. If a "class" has no state, it should probably be a module of plain functions instead.

### Encapsulation
Hide internal state; expose behavior through a clear interface. Don't let external code reach in and mutate internals directly.

```python
# Bad: internal state is public and can be set to anything, anywhere
class Population:
    def __init__(self, size):
        self.infected = 0
        self.size = size

pop = Population(1000)
pop.infected = -50  # nonsensical state, nothing stops it

# Good: state changes go through methods that enforce invariants
class Population:
    def __init__(self, size: int):
        self._size = size
        self._infected = 0

    def infect(self, n: int) -> None:
        self._infected = min(self._size, self._infected + n)

    @property
    def infected(self) -> int:
        return self._infected
```

### Abstraction
Expose *what* something does, hide *how*. Callers of `Simulation.run()` shouldn't need to know it's using RK4 internally vs. Euler's method.

### Inheritance
Use it to model genuine "is-a" relationships with shared behavior — not to share code between unrelated classes. If you're inheriting just to reuse a couple of methods, prefer composition instead.

```python
# Bad: inheriting just to reuse a helper method, no real "is-a" relationship
class CSVWriterMixin:
    def to_csv(self, path): ...

class SIRModel(CSVWriterMixin):  # a model "is-a" CSV writer? no.
    ...

# Good: compose instead
class SIRModel:
    def __init__(self, writer: CSVWriter):
        self._writer = writer
```

### Polymorphism
Code against a common interface so different implementations are interchangeable, enabling the Open/Closed and Dependency Inversion examples above (`EpidemicModel`, `DataLoader`, etc.).

## 8. Unit Testing: Not Optional

If it isn't tested, it doesn't work — you just haven't found the input that breaks it yet. Untested code in a simulation/analysis project is especially dangerous because a wrong number often looks exactly as plausible as a right one; nothing crashes, nothing looks off, and it ends up in a plot or a report.

**This is how §1's "small composed parts" idea actually gets enforced.** A "unit" test is called that because it tests one *unit* — the smallest coherent piece of code (a function, a class method). Composing a system out of small parts is only a good idea if you can prove each part is correct in isolation, independent of everything it's wired up to; otherwise you're just assembling unverified pieces and hoping. Unit testing *is* the act of verifying each small part works on its own, and that discipline is exactly what makes it safe to trust the composition — a system built from many well-tested small parts, before it's asked to also be correct as a whole.

**Why it's a MUST, not a nice-to-have:**

- **It's the only thing that actually catches regressions.** Comments and docstrings describe intent; they can't verify it, and as established in §5, they rot silently. A test either passes or it doesn't — it can't lie to you the way a stale comment can.
- **It's what makes refactoring safe.** Every reorganization promised in §2/§3 (extracting duplicated logic, splitting a god-class by SRP, swapping an implementation via DIP) is only safe if a test suite proves behavior didn't change. Without tests, "refactor" is just "rewrite and hope."
- **It forces better design, not just better safety.** Code that's hard to unit test is almost always a design smell: hidden global state, a class doing five things (§3, SRP), a hard-wired dependency instead of an injected one (§3, DIP). Writing the test is often what surfaces the fix.
- **It's the fastest feedback loop that exists.** Finding a bug via a 2-second test run is cheap. Finding the same bug because a teammate's plot looks wrong three days before the deadline is expensive, and often traced back to a subtle math/indexing error, exactly the kind unit tests exist to catch.
- **In a group project, it's how you trust code you didn't write.** You should not have to read and mentally execute a teammate's function to trust it — you should be able to look at its tests, see the behaviors they assert, and trust the green checkmark.

**How to do it:**

- We use `pytest`. Test files live under `tests/`, mirroring the `src/` structure (`src/models/sir.py` → `tests/models/test_sir.py`), named `test_*.py`, with test functions named `test_*`.
- **One test, one behavior.** Don't cram five assertions about unrelated behaviors into one test function — when it fails, the name (`test_infect_clamps_at_population_size`) should already tell you what broke, without reading the body.
- **Arrange – Act – Assert.** Set up inputs, call the thing, assert the outcome. Keep those three steps visually distinguishable.
- **Test behavior, not implementation.** Test what a function returns/does for given inputs, not its internal steps — otherwise every internal refactor breaks tests that were never actually testing anything meaningful.
- **Cover edge cases, not just the happy path**: empty input, zero, negative values, boundary values (e.g. `infected == size`), and at least one case that should raise.
- **For numeric/scientific code, assert with tolerance**, never exact float equality — use `pytest.approx` or `np.isclose`, since floating-point math almost never lands on an exact value.
- **Design for testability using DIP (§3).** Inject dependencies (data loaders, random seeds, I/O) so tests can substitute fakes/fixtures instead of hitting real files, networks, or non-deterministic randomness.
- Every new function/class gets tests in the *same* PR that introduces it — not "added later." Tests added later tend to never get added.
- CI must pass before merge. A red test suite blocks the PR, no exceptions.

```python
# src/models/population.py
class Population:
    def __init__(self, size: int):
        self._size = size
        self._infected = 0

    def infect(self, n: int) -> None:
        self._infected = min(self._size, self._infected + n)

    @property
    def infected(self) -> int:
        return self._infected


# tests/models/test_population.py
import pytest
from src.models.population import Population

def test_infect_increases_infected_count():
    pop = Population(size=100)
    pop.infect(10)
    assert pop.infected == 10

def test_infect_clamps_at_population_size():
    pop = Population(size=100)
    pop.infect(500)
    assert pop.infected == 100

def test_infect_accumulates_across_multiple_calls():
    pop = Population(size=100)
    pop.infect(30)
    pop.infect(30)
    assert pop.infected == 60
```

```python
# Numeric code: never assert exact floats
def test_mse_matches_expected_value():
    predictions = np.array([1.0, 2.0, 3.0])
    actual = np.array([1.1, 1.9, 3.2])
    assert compute_mse(predictions, actual) == pytest.approx(0.02, abs=1e-4)
```

## 9. Before You Open a PR, Ask Yourself

- Could a teammate understand this function from its name and signature alone?
- If I deleted every comment, would the code still make sense? For any comment I kept, would it still be true after the *next* change someone makes here — or is it already at risk of drifting?
- Is there duplicated logic I should extract?
- Does this class have more than one reason to change?
- Would adding a new model/data source/metric require editing existing, working code — or just adding a new piece?
- Is there a `for` loop over a DataFrame/array that should be a vectorized `numpy`/`pandas` operation instead?
- If this touches a dataset that doesn't fit in memory, should this be Spark instead of pandas?
- Did I write tests in this same PR for every new function/class? Do they cover edge cases, not just the happy path?
- If this is numeric code, are float assertions using a tolerance (`pytest.approx`/`np.isclose`) instead of exact equality?
- Does CI pass?

Good code isn't clever code. It's code that disappears into the background because it's obviously correct.
