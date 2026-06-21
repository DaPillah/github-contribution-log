# Contribution #1: Surface dbt's stdout/stderr in CosmosLoadDbtException when the dbt command fails with no output #2822

**Contribution Number:** 1  
**Student:** Justin  
**Issue:** [GitHub issue link](https://github.com/astronomer/astronomer-cosmos/issues/2822)  
**Status:** Phase IV [In Progress]

---

## Why I Chose This Issue

I chose this issue because it involves improving error handling in a real production Python 
codebase used by data engineers working with dbt and Apache Airflow. As someone interested in 
data engineering and AI/ML workflows, working on Astronomer's Cosmos project gives me direct 
exposure to how production data pipelines handle failures and surface useful debugging information.

As a first-time open source contributor through CodePath, I wanted to work on something that 
would teach me meaningful Python patterns. This issue specifically helps me learn how subprocess 
output is captured and surfaced in exception messages, which is a skill directly applicable to any 
backend Python work.

---

## Understanding the Issue

### Problem Description

When Cosmos runs a dbt command via `LoadMode.DBT_LS` and it fails, the error message is often 
opaque (especially when both stdout and stderr appear empty), making it very difficult to diagnose 
why the command failed.

### Expected Behavior

When a dbt command fails, `CosmosLoadDbtException` should include the captured stdout, stderr, 
and process exit code in its message (even explicitly stating when no output was captured) so 
developers can quickly understand what went wrong.

### Current Behavior

The exception only shows `Unable to run [... dbt ls ...]` with little or no detail, leaving 
developers without enough information to debug the failure.

### Affected Components

- **`cosmos/dbt/graph.py`** — specifically the `run_command` and `run_command_with_subprocess` 
  functions where the exception is raised

---

## Reproduction Process

### Environment Setup

Set up the local development environment using `hatch`, the project's test runner. The main 
challenge was understanding the hatch matrix environment naming convention 
(e.g. `tests.py3.11-2.10-1.9:test`). These aren't discoverable by guessing and are documented 
in `AGENTS.md`. Once the correct command was found, the environment bootstrapped cleanly.

### Steps to Reproduce

1. Clone the repo and set up the environment with `hatch`
2. Run `hatch run tests.py3.11-2.10-1.9:test tests/dbt/test_graph.py::test_run_command_none_argument`
3. Observed: test fails because `CosmosLoadDbtException` message does not include exit code, 
   stderr, or stdout, making it impossible to diagnose what went wrong

### Reproduction Evidence

- **log:**  
  <img width="832" height="309" alt="GithubC1SS" src="https://github.com/user-attachments/assets/c248573c-5034-47df-a762-07d22e3f8d71" />
- **My findings:** `run_command_with_subprocess` raised `CosmosLoadDbtException` with only a 
  generic message — no exit code included, and the error detection condition `if returncode or ...` 
  silently swallowed `None` exit codes as falsy successes

---

## Solution Approach

### Analysis

Two problems in `run_command_with_subprocess` in `cosmos/dbt/graph.py`:

1. The error message format did not include the exit code, making it hard to diagnose failures
2. The condition `if returncode or "Error" in stdout` treated `None` as falsy (success), so 
   commands returning a `None` exit code would silently pass instead of raising an error

### Proposed Solution

Add `Exit code`, `stderr`, and `stdout` to all `CosmosLoadDbtException` messages in 
`run_command_with_subprocess`, and change the error-detection condition to `returncode != 0` 
so `None` is correctly treated as an undefined/failed result.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `CosmosLoadDbtException` was raised without surfacing the dbt process output, 
making it hard for users to debug failures. The exit code was also not included.

**Match:** The `run_command_with_dbt_runner` function in the same file already follows a pattern 
of building a `details` string with stderr and stdout before raising — I followed the same pattern 
and added exit code to it.

**Plan:**
1. Modify `run_command_with_subprocess` in `cosmos/dbt/graph.py` to include `Exit code`, `stderr`, 
   and `stdout` in both error paths
2. Change `if returncode or ...` to `if returncode != 0 or ...`
3. Update `test_run_command` success cases from `returncode=None` to `returncode=0`
4. Add `mock_popen.return_value.returncode = None` to `test_run_command_none_argument` and update 
   its expected string

**Implement:** https://github.com/astronomer/astronomer-cosmos/pull/2826/commits

**Review:**
- Follows project line length of 120 chars ✅
- No new logging calls needed ✅
- Commit message uses imperative mood, no Conventional Commits prefix ✅
- No `Co-Authored-By` trailer for AI ✅

**Evaluate:** Run `hatch run tests.py3.11-2.10-1.9:test tests/dbt/test_graph.py` and confirm 
all 1701 tests pass with 0 failures

---

## Testing Strategy

### Unit Tests

- [x] `test_run_command_none_argument`: verifies exit code, stderr, and stdout appear in the 
      exception message when a command with a `None` argument fails
- [x] `test_run_command[all good-0]`: verifies a zero exit code with clean stdout does not raise
- [x] `test_run_command[WarnErrorOptions-0]`: verifies `WarnErrorOptions` in stdout is not 
      mistaken for an error
- [x] `test_run_command[fail-599]`: verifies a non-zero exit code raises `CosmosLoadDbtException`
- [x] `test_run_command[Error-None]`: verifies `"Error"` in stdout raises `CosmosLoadDbtException`

### Integration Tests

- [ ] Not required for this change — no new behavior is introduced at the integration level

### Manual Testing

Ran the full `test_graph.py` test suite with 
`hatch run tests.py3.11-2.10-1.9:test tests/dbt/test_graph.py`. 
Result: **1701 passed, 18 skipped, 2 xfailed** — no regressions.

---

## Implementation Notes

### Code Changes

- **Files modified:** `cosmos/dbt/graph.py`, `tests/dbt/test_graph.py`
- **Key commits:** [Link to your commit]
- **Approach decisions:**
  - Changed to `returncode != 0` instead of adding a separate `None` check — cleaner and 
    semantically correct since `0` is the only valid success exit code for a subprocess
  - Kept `stderr` before `stdout` in the generic error path to match the order expected by 
    the existing test
  - Updated `test_run_command` success cases to use `returncode=0` rather than `None` since 
    `0` is the real success exit code — `None` was only working before because it happened 
    to be falsy


---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
