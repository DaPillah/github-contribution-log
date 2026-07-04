# My Open Source Contributions
---
---


## Contribution #2: Default ModuleConfig args not captured in wandb config parameters #596
**Student:** Justin  
**Issue:** [GitHub issue link](https://github.com/ai2cm/ace/issues/596)
**Status:** Phase III [In Progress]

---

## Why I Chose This Issue

I chose this issue because it sits at the intersection of Python internals 
and ML experiment tracking, which are two areas I want to build real skills in. 
Unlike my first contribution which focused on error handling, this one 
requires understanding how Python dataclasses handle default values and 
how ML teams use Weights & Biases (wandb) to track model configs. It's 
a meaningful step up that introduces me to research-grade ML codebases.

---

## Understanding the Issue

### Problem Description

When a ModuleConfig dataclass is logged to wandb, only values explicitly 
set in the yaml config file are captured. Default field values defined in 
the dataclass itself are silently omitted from the wandb config parameters.

### Expected Behavior

All config values (both explicitly set and default) should appear in 
the wandb config so that experiment runs are fully reproducible and 
comparable, even if defaults change in the future.

### Current Behavior

Only values written in the yaml file are logged to wandb. If a field 
like `pad` has a default value in the dataclass but isn't mentioned in 
the yaml, it won't appear in the wandb run overview at all.


### Affected Components

- `ModuleConfig` dataclass and its subclasses
- The wandb config logging logic in the training pipeline
- yaml serialization/deserialization utilities

---

## Reproduction Process

### Environment Setup
- Forked and cloned the ai2cm/ace repository
- Created conda environment using `make create_environment`
- Activated environment with `conda activate fme`
- Installed pre-commit hooks with `pre-commit install`
- Confirmed baseline tests pass with `make test_very_fast`

### Steps to Reproduce
1. Write a yaml config file for a ModuleConfig subclass that omits 
   fields which have default values defined in the dataclass
2. Run a training job that logs the config to wandb
3. Open the wandb run overview and search for the omitted field (e.g. `pad`)
4. Observe that the field is missing from the Config parameters section 
   entirely, even though the model is actively using the default value

### Reproduction Evidence
- **Log:** Maintainer-provided wandb run (private/restricted access):
  https://wandb.ai/ai2cm/ace-samudra-cm4/runs/4nrk3kev/overview
  Note: This run is not publicly accessible, but was referenced by the 
  maintainer in the issue thread to demonstrate the missing default values.

- **My findings:** The bug originates from how wandb config is populated.
  The raw yaml dict is logged directly, which only contains values the 
  user explicitly wrote. Default values defined in the ModuleConfig 
  dataclass are never written to yaml in the first place, so they are 
  invisible to wandb. The fix is to instantiate the dataclass from the 
  raw yaml first (which auto-populates defaults), convert it back to yaml, 
  then log that to wandb instead.
  
---
---



## Contribution #1: `Surface dbt's stdout/stderr in CosmosLoadDbtException when the dbt command fails with no output #2822` 
**Student:** Justin  
**Issue:** [GitHub issue link](https://github.com/astronomer/astronomer-cosmos/issues/2822)  
**Status:** Phase IV [Merged ✅]

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
all 1703 tests pass with 0 failures

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
- [x] `test_run_command_surfaces_stderr_in_exception`: verifies stderr content appears in the 
      exception message and `<no stdout captured>` placeholder shows when stdout is empty
- [x] `test_run_command_surfaces_stdout_in_exception`: verifies stdout content appears in the 
      exception message and `<no stderr captured>` placeholder shows when stderr is empty

### Integration Tests

- [x] `test_load_via_dbt_ls_with_non_zero_returncode`: updated regex to include `Exit code: 1` 
      and `<no stdout captured>` to match new message format
- [x] `test_load_via_dbt_ls_with_runtime_error_in_stdout`: updated regex to include `Exit code: .*` 
      and `<no stderr captured>` to match new message format
- [x] `test_load_via_dbt_ls_without_dbt_deps`: updated assertion from `==` to `startswith` 
      since the message now includes variable exit code and stdout after the base message

### Manual Testing

Ran the full `test_graph.py` test suite with 
`hatch run tests.py3.11-2.10-1.9:test tests/dbt/test_graph.py`. 
Result: **1703 passed, 18 skipped, 2 xfailed** — no regressions.

---

## Implementation Notes

### Code Changes

- **Files modified:** `cosmos/dbt/graph.py`, `tests/dbt/test_graph.py`
- **Key commits:** 
  - https://github.com/astronomer/astronomer-cosmos/pull/2826/changes/8eae902d8c10e6916f7d817b60ab99aa0bc6e787
  - https://github.com/astronomer/astronomer-cosmos/pull/2826/changes/1ccb0fef0f3e1acebe551b0211faf46f61236c63
  - https://github.com/astronomer/astronomer-cosmos/pull/2826/changes/b017b3fd64e38fc7948b26b8aaf3b160188a8b8b
- **Approach decisions:**
  - Changed to `returncode != 0` instead of adding a separate `None` check — cleaner and 
    semantically correct since `0` is the only valid success exit code for a subprocess
  - Kept `stderr` before `stdout` in the generic error path to match the order expected by 
    the existing test
  - Updated `test_run_command` success cases to use `returncode=0` rather than `None` since 
    `0` is the real success exit code — `None` was only working before because it happened 
    to be falsy
  - Added `stdout = stdout or "<no stdout captured>"` and `stderr = stderr or 
    "<no stderr captured>"` fallbacks after `process.communicate()` per maintainer feedback 
    — makes empty output explicit instead of rendering blank
  - Normalized both error paths in `run_command_with_subprocess` to use the same format: 
    `Exit code / stderr / stdout`
  - Updated existing integration test regex patterns to match the new message format
  - Added two new tests covering empty stderr and empty stdout scenarios per maintainer request
  - Changed `test_load_via_dbt_ls_without_dbt_deps` assertion from `==` to `startswith` 
    since the dbt_packages error message now appends variable exit code and stdout content 
    that differs across CI environments

---

## Pull Request

**PR Link:** https://github.com/astronomer/astronomer-cosmos/pull/2826

**PR Description:** Include exit code, stderr, and stdout in all CosmosLoadDbtException 
messages raised by run_command_with_subprocess. Also tighten the error-detection condition 
from truthy-returncode to returncode != 0, so that a None exit code (undefined result) is 
correctly treated as a failure rather than silently swallowed.

Closes #2822

**Maintainer Feedback:**
- June 23, 2026: @tatiana requested four changes — (1) add explicit placeholders when 
  stderr/stdout are empty, (2) normalize the error message format between both error paths 
  in graph.py, (3) update integration test regex patterns to match the new message format, 
  (4) add new pytest.raises test cases verifying the new error message format for both 
  empty and non-empty stdout/stderr scenarios
- June 24, 2026: Addressed all feedback — added `stdout or "<no stdout captured>"` and 
  `stderr or "<no stderr captured>"` fallbacks, normalized both error paths to use the same 
  format, updated existing test regexes, and added two new tests 
  (test_run_command_surfaces_stderr_in_exception and 
  test_run_command_surfaces_stdout_in_exception)
- June 29, 2026: @tatiana flagged remaining integration test failures from CI. Identified 
  two separate issues:
  - `test_load_via_dbt_ls_without_dbt_deps` was doing an exact string match on a message 
    that now has exit code and stdout appended; fixed by changing `==` to `startswith`
  - Three `test_runner.py` failures were pre-existing CI infrastructure issues with 
    `dbt.cli.main` not being available in that environment, unrelated to this PR

**Status:** Merged ✅ — tatiana merged into `astronomer:main` on June 30, 2026.  
144 checks passed. Code coverage: 98.44% (all modified lines covered).

---

## Learnings & Reflections

### Technical Skills Gained

- How subprocess output (stdout, stderr, exit codes) is captured and surfaced in Python 
  exception messages
- How to read and follow contributing guidelines (`AGENTS.md`, `CLAUDE.md`) in a real 
  production open source project
- How hatch matrix environments work for running tests across multiple Python/Airflow versions
- How CI pipelines work for external contributors and why integration tests require special 
  permissions to run
- How to iterate on a PR based on maintainer feedback. Responding to inline comments, 
  pushing follow-up commits, and communicating clearly about what was changed and why

### Challenges Overcome

- Understanding why `returncode = None` was treated as success by the original `if returncode or ...` 
  condition — required tracing through Python's truthiness rules and subprocess behavior
- Figuring out why the test mock needed `mock_popen.return_value.returncode = None` explicitly 
  set. MagicMock auto-creates child mocks for unset attributes, which wouldn't format as `None`
- Identifying which CI failures were caused by our changes vs pre-existing infrastructure issues
- Understanding the difference between `git stash`, `git rebase`, and why commit order matters 
  when syncing with a remote branch

### What I'd Do Differently Next Time

- Pull the latest `main` before starting each new round of changes to avoid diverged branch issues
- Run the full test suite locally before pushing to catch integration test regressions earlier
- Read `AGENTS.md` more carefully upfront (the hatch environment naming convention cost time 
  to discover).

---

## Resources Used

- [Astronomer Cosmos AGENTS.md](https://github.com/astronomer/astronomer-cosmos/blob/main/AGENTS.md)
- [Issue #2822](https://github.com/astronomer/astronomer-cosmos/issues/2822)
- [PR #2826](https://github.com/astronomer/astronomer-cosmos/pull/2826)
- [Python subprocess docs](https://docs.python.org/3/library/subprocess.html)
- [Python unittest.mock docs](https://docs.python.org/3/library/unittest.mock.html)
