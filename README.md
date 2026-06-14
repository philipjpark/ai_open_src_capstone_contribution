# ai_open_src_capstone_contribution

## Contribution #1: Add `approximation_degree` argument to `CommutativeCancellation` pass and set it in the preset

**Contribution Number:** 1
**Student:** Philip Park
**Issue:** [Qiskit/qiskit#14115](https://github.com/Qiskit/qiskit/issues/14115)
**Status:** Phase II Complete

---

## Why I Chose This Issue

This issue adds an `approximation_degree` parameter to the `CommutativeCancellation` optimization pass so that approximation settings can properly flow through the internal optimization pipeline instead of being hidden inside lower-level analysis code.

It matters because modern optimization systems, whether in quantum computing, AI inference, or quantitative trading infrastructure, rely on configurable tradeoffs between precision, speed, and computational efficiency. This issue improves how those controls propagate through a larger execution graph.

I chose this issue because as someone interested in quant trading and building my own ML models, I want deeper experience understanding optimization pipelines, DAG-based execution systems, and how real-world infrastructure exposes tunable parameters for balancing accuracy versus performance.

---

## Understanding the Issue

### Problem Description

`CommutativeCancellation` is a transpiler optimization pass that cancels redundant gates using commutation relations. Internally, it calls the Rust function `cancel_commutations`, which uses commutation analysis with an `approximation_degree` parameter.

However, the Python wrapper for `CommutativeCancellation` does not currently expose `approximation_degree` in its constructor, does not forward it in `run()`, and the preset pass managers do not pass `pass_manager_config.approximation_degree` when constructing the pass.

As a result, when a user calls `transpile(..., approximation_degree=0.8)`, that setting reaches some optimization passes but not `CommutativeCancellation`.

### Expected Behavior

* `CommutativeCancellation` should accept an `approximation_degree` argument.
* Preset pass managers should pass `pass_manager_config.approximation_degree` when constructing `CommutativeCancellation`.
* Setting `approximation_degree` in `transpile()` should affect commutation analysis inside the cancellation pass.

### Current Behavior

* `CommutativeCancellation.__init__` only accepts `basis_gates` and `target`.
* `run()` calls `cancel_commutations(dag, checker, basis)` without forwarding `approximation_degree`.
* In `builtin_plugins.py`, multiple call sites use `CommutativeCancellation()` or `CommutativeCancellation(target=...)` without passing `approximation_degree`.

### Affected Components

| Component                  | Path                                                                |
| -------------------------- | ------------------------------------------------------------------- |
| Python pass wrapper        | `qiskit/transpiler/passes/optimization/commutative_cancellation.py` |
| Rust implementation        | `crates/transpiler/src/passes/commutation_cancellation.rs`          |
| Commutation analysis       | `crates/transpiler/src/passes/commutation_analysis.rs`              |
| Preset pass manager wiring | `qiskit/transpiler/preset_passmanagers/builtin_plugins.py`          |
| Existing tests             | `test/python/transpiler/test_commutative_cancellation.py`           |

---

## Reproduction Process

### Environment Setup

**OS:** Windows 10
**Python:** 3.12.5
**Fork:** https://github.com/philipjpark/qiskit
**Working branch:** https://github.com/philipjpark/qiskit/tree/fix-issue-14115

#### Setup steps completed

1. Cloned my fork locally.
2. Added the upstream Qiskit remote.
3. Created a Python virtual environment at `.venv` in the repository root.
4. Upgraded pip.
5. Installed Rust using rustup.
6. Installed Qiskit in the virtual environment for reproduction and API inspection.

#### Commands used

```powershell
cd C:\Users\phili\Desktop\qiskit
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -U pip
.\.venv\Scripts\pip.exe install qiskit
.\.venv\Scripts\pip.exe install --group dev
```

#### Setup challenges

| Problem                                                            | Fix / Workaround                                                                             |
| ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| Rust toolchain was partially installed and `cargo.exe` was missing | Reinstalled the Rust toolchain using rustup                                                  |
| `pip install -e .` failed during Rust compilation                  | Documented the issue for Phase III and used installed Qiskit for reproduction/API inspection |
| Source build is slow on Windows because of Rust crates             | Will use the Qiskit build instructions and retry editable install during Phase III           |

For Phase III source development, I plan to retry with:

```powershell
rustup toolchain install 1.87-x86_64-pc-windows-msvc
$env:RUSTUP_AUTO_INSTALL = "0"
$env:RUSTUP_TOOLCHAIN = "1.87-x86_64-pc-windows-msvc"
.\.venv\Scripts\pip.exe install --group build
.\.venv\Scripts\pip.exe install -e .
```

---

### Steps to Reproduce

1. Activate the virtual environment.

```powershell
.\.venv\Scripts\Activate.ps1
```

2. Run the reproduction script.

```powershell
.\.venv\Scripts\python.exe scripts\repro_issue_14115.py
```

3. Confirm that `CommutativeCancellation.__init__` does not include `approximation_degree`.

Observed output:

```text
CommutativeCancellation.__init__ parameters:
['self', 'basis_gates', 'target']

Has approximation_degree: False
```

4. Compare this with a sibling optimization pass such as `RemoveIdentityEquivalent`.

Observed output:

```text
RemoveIdentityEquivalent.__init__
Has approximation_degree: True
```

5. Confirm that the Rust binding already supports `approximation_degree`.

Observed output:

```text
cancel_commutations parameters:
['dag', 'commutation_checker', 'basis_gates', 'approximation_degree']
```

6. Inspect `qiskit/transpiler/passes/optimization/commutative_cancellation.py`.

Observed result:

```python
cancel_commutations(dag, checker, basis)
```

The call does not forward `approximation_degree`.

7. Inspect `qiskit/transpiler/preset_passmanagers/builtin_plugins.py`.

Observed result:

* Neighboring passes pass `pass_manager_config.approximation_degree`.
* `CommutativeCancellation` is constructed without passing `approximation_degree`.

8. Repeat the API/source inspection to confirm the issue is deterministic.

---

### Reproduction Evidence

**Working branch in my fork:**
https://github.com/philipjpark/qiskit/tree/fix-issue-14115

**Script output:**

```text
1. API gap: CommutativeCancellation.__init__ parameters
   ['self', 'basis_gates', 'target']
   Has approximation_degree: False

2. Sibling pass: RemoveIdentityEquivalent.__init__
   Has approximation_degree: True

3. CommutativeCancellation.run() forwards approximation_degree: False

4. Rust binding cancel_commutations parameters:
   ['dag', 'commutation_checker', 'basis_gates', 'approximation_degree']
```

**My findings:**

* The bug is an API/plumbing issue, not a missing Rust implementation.
* The Rust layer already supports `approximation_degree`.
* The Python pass wrapper and preset pass manager wiring are the missing pieces.
* This issue can be reproduced by inspecting the constructor signature, the `run()` method, and the preset pass manager call sites.

---

## Solution Approach

### Analysis

The root cause is that `approximation_degree` support exists lower in the commutation cancellation pipeline, but the public Python pass wrapper does not expose or forward that parameter.

The expected flow should be:

```text
transpile(approximation_degree=X)
  -> PassManagerConfig.approximation_degree
  -> CommutativeCancellation(approximation_degree=X)
  -> cancel_commutations(..., approximation_degree=X)
  -> analyze_commutations(..., approximation_degree=X)
```

Currently, the middle step is missing.

---

### Proposed Solution

Add `approximation_degree` to the `CommutativeCancellation` constructor, store it on the pass instance, forward it to `cancel_commutations()` inside `run()`, and update the preset pass managers so they pass `pass_manager_config.approximation_degree` when constructing `CommutativeCancellation`.

---

### Implementation Plan

Using the UMPIRE framework:

#### Understand

Users expect `approximation_degree` passed to `transpile()` to affect optimization behavior throughout the transpiler pipeline. `CommutativeCancellation` currently ignores this setting because the parameter is not exposed or forwarded.

#### Match

Similar patterns already exist in the codebase:

| Reference                  | Pattern to follow                                                                            |
| -------------------------- | -------------------------------------------------------------------------------------------- |
| `RemoveIdentityEquivalent` | Constructor accepts `approximation_degree`, stores it, and passes it into lower-level logic  |
| `CommutativeOptimization`  | Similar optimization pass that accepts `approximation_degree`                                |
| `builtin_plugins.py`       | Other passes receive `pass_manager_config.approximation_degree` from the preset pass manager |

#### Plan

1. Modify `qiskit/transpiler/passes/optimization/commutative_cancellation.py`.

   * Add `approximation_degree: float | None = 1.0` to `__init__`.
   * Store it as `self._approximation_degree`.
   * Update `run()` so it forwards the value into `cancel_commutations`.

2. Modify `qiskit/transpiler/preset_passmanagers/builtin_plugins.py`.

   * Update all `CommutativeCancellation(...)` call sites.
   * Pass `approximation_degree=pass_manager_config.approximation_degree`.

3. Add or update tests in `test/python/transpiler/test_commutative_cancellation.py`.

   * Confirm `CommutativeCancellation(approximation_degree=0.5)` works.
   * Confirm default behavior still works.
   * Confirm the pass can run with a non-default approximation degree.

4. Add a release note if required by the Qiskit contribution guidelines.

5. Run the relevant test suite before submitting a pull request in Phase III.

---

### Implement

Implementation will happen in Phase III.

Current working branch:
https://github.com/philipjpark/qiskit/tree/fix-issue-14115

---

### Review

Before submitting a pull request, I will check:

* Qiskit contribution guidelines
* Existing code style
* Existing test patterns
* Whether a release note is required
* Whether the PR description includes `Fixes #14115`
* Whether I need to disclose AI tool usage
* Whether the CLA is signed before merge

---

### Evaluate

To verify the fix, I will:

* Re-run the reproduction script and confirm `CommutativeCancellation.__init__` includes `approximation_degree`.
* Re-run the reproduction script and confirm `run()` forwards the parameter.
* Run existing `test_commutative_cancellation.py` tests.
* Add or update tests for non-default `approximation_degree`.
* Run a basic transpile integration test with `approximation_degree=0.5`.

---

## Testing Strategy

### Unit Tests

* [ ] `CommutativeCancellation(approximation_degree=0.5)` instantiates without error.
* [ ] `CommutativeCancellation()` still works with the default value.
* [ ] Existing `test_commutative_cancellation.py` tests still pass.
* [ ] A non-default approximation degree is forwarded into the lower-level cancellation call.

### Integration Tests

* [ ] `transpile(circuit, approximation_degree=0.5, optimization_level=2)` runs successfully.
* [ ] Preset pass managers construct `CommutativeCancellation` with `pass_manager_config.approximation_degree`.

### Manual Testing

* [ ] Run `scripts/repro_issue_14115.py` after the fix and confirm `Has approximation_degree: True`.
* [ ] Run the relevant Qiskit transpiler tests from the repository root.

---

## Implementation Notes

### Week 2 Progress: Phase II

* Set up a Windows local development environment.
* Created a Python virtual environment.
* Installed Rust tooling.
* Created and pushed the branch `fix-issue-14115` to my fork.
* Reproduced the issue through API inspection and source code review.
* Identified the root cause.
* Wrote a UMPIRE-based implementation plan.
* Documented source build challenges for Phase III.

### Week 3+ Progress

To be filled during Phase III implementation.

---

## Code Changes

### Files expected to change in Phase III

* `qiskit/transpiler/passes/optimization/commutative_cancellation.py`
* `qiskit/transpiler/preset_passmanagers/builtin_plugins.py`
* `test/python/transpiler/test_commutative_cancellation.py`
* `releasenotes/notes/...`

### Key commits

Pending Phase III.

### Approach decisions

I will follow existing `RemoveIdentityEquivalent` and `CommutativeOptimization` patterns rather than inventing new API behavior. The goal is to make `CommutativeCancellation` consistent with sibling optimization passes that already receive `approximation_degree`.

---

## Pull Request

**PR Link:** Pending Phase III.

**PR Description draft:**

```text
Fixes #14115

Adds an approximation_degree parameter to CommutativeCancellation and wires it through the preset pass managers so that the transpiler-level approximation setting reaches commutation analysis inside the cancellation pass.
```

**Maintainer Feedback:** Pending.

**Status:** Not yet submitted.

---

## Learnings & Reflections

### Technical Skills Gained

* Navigated a hybrid Python/Rust codebase.
* Learned how Qiskit's transpiler passes are wired through preset pass managers.
* Practiced reproducing an infrastructure/plumbing issue through API and source inspection.
* Troubleshot Windows Rust toolchain setup.

### Challenges Overcome

* Rust toolchain installation issue.
* Source build complexity on Windows.
* Understanding the difference between the issue description and the current Rust-internal architecture.

### What I'd Do Differently Next Time

* Check for existing open PRs earlier.
* Start editable install earlier to allow more time for Rust build troubleshooting.

---

## Resources Used

* [Qiskit #14115](https://github.com/Qiskit/qiskit/issues/14115)
* [Qiskit CONTRIBUTING.md](https://github.com/Qiskit/qiskit/blob/main/CONTRIBUTING.md)
* [Rustup Windows MSVC docs](https://rust-lang.github.io/rustup/installation/windows-msvc.html)
