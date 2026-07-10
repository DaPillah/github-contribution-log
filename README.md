# My Open Source Contributions
---
---


## Contribution #2: Default ModuleConfig args not captured in wandb config parameters #596
**Student:** Justin  
**Issue:** [GitHub issue link](https://github.com/ai2cm/ace/issues/596)   
**Status:** Phase IV [Merged ✅]

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

## Solution Approach

### Analysis

`ModuleSelector` (`fme/core/registry/module.py`) is a dataclass that stores a network builder as `type: str` + `config: Mapping[str, Any]`, where `config` is the raw dict loaded from yaml. In `__post_init__` it builds the real `ModuleConfig` subclass into `self._instance` (this is where Python fills in defaults like `pad`). However:

- `self._instance` is a plain attribute, **not** a dataclass field, so `dataclasses.asdict()` ignores it.
- `self.config` **is** a field, so `asdict()` serializes it, but it is still the raw yaml dict, missing defaults.

When a run is logged, the caller does `dataclasses.asdict(TrainConfig)` (`fme/ace/train/train.py`), which serializes the incomplete `self.config`. The fully-defaulted object exists but is never serialized. This is why defaults never reach wandb.

### Proposed Solution

Normalize `ModuleSelector.config` inside `__post_init__`: immediately after building `self._instance`, overwrite `self.config` with `dataclasses.asdict(self._instance)`. This replaces the raw yaml dict with the fully-defaulted version, so defaults are captured wherever the config is serialized (wandb, checkpoints, logs), not just wandb.

### Implementation Plan

Using the UMPIRE framework (adapted):

- **Understand:** wandb only shows user-specified config values; `ModuleConfig` defaults are missing, which hurts reproducibility and run comparison. Reproduced via `SamudraBuilder.pad`.
- **Match:** This is a serialization-gap problem. The repo already uses `dataclasses.asdict(selector.module_config)` elsewhere (`test_module_registry.py`), so the "instantiate → asdict" pattern is idiomatic here and known to serialize cleanly.
- **Plan:**
  1. Add a failing test: a mock builder with a default field; assert the default appears in `dataclasses.asdict(selector)["config"]`.
  2. Fix `ModuleSelector.__post_init__` to normalize `self.config`.
  3. Confirm the test goes red → green.
  4. Run the registry and module-builder/step suites to check for regressions.
  5. Run pre-commit (ruff, ruff-format, mypy).
- **Implement:** One line added to `__post_init__` — `self.config = dataclasses.asdict(self._instance)` — plus a comment citing #596.
- **Review:**
  - Verified loading still works: `ModuleConfig.from_state` uses `dacite(strict=True)`, and every added key is a valid field, so both older (partial) and newer (full) configs load.
  - Checkpoint backwards-compatibility tests (`test_frozen_module_backwards_compatibility`, `test_latest_module_backwards_compatibility`) still pass.
- **Evaluate:** The fix is minimal (one line), addresses the root cause at the correct layer, and fixes all serialization consumers rather than patching wandb alone.

---

## Testing Strategy

### Unit Tests

- Added `MockModuleBuilderWithDefault` (a `ModuleConfig` subclass with a required `param_shapes` field and a defaulted `pad: str = "reflect"` field) and `test_module_selector_config_includes_defaults` in `fme/core/registry/test_module_registry.py`. The test builds a `ModuleSelector` supplying only `param_shapes`, then asserts `dataclasses.asdict(selector)["config"]["pad"] == "reflect"`.
- **Red before fix:** `KeyError: 'pad'` (default genuinely missing).
- **Green after fix:** passes.

### Integration Tests

- Ran the full registry suite (`fme/core/registry/`): **8 passed**, including the frozen/latest checkpoint backwards-compatibility tests.
- Ran the module-builder and step suites (`fme/ace/registry/`, `fme/core/step/test_step.py`, `fme/core/models/mlp/test_mlp.py`): **210 passed, 7 skipped** — no regressions.

### Manual Testing

I do not have access to the maintainer's private wandb project, so I verified at the serialization layer that produces the wandb config: constructing a `ModuleSelector` with only the required fields and confirming `dataclasses.asdict(selector)["config"]` now contains the default field (which `wandb.init(config=...)` ultimately receives). Also ran `pre-commit run --files ...` — ruff, ruff-format, and mypy all pass.

---

## Implementation Notes

### Code Changes

- **Files modified:**
  - `fme/core/registry/module.py` — normalize `config` in `ModuleSelector.__post_init__`.
  - `fme/core/registry/test_module_registry.py` — add mock builder with a default field and a failing-first test.
- **Key commits:**
  - `9d121dc1` — *Serialize ModuleConfig defaults in ModuleSelector.config* (fix + test together, one reviewable unit).
- **Approach decisions:**
  - Fixed in `ModuleSelector.__post_init__`, not in the wandb-logging code — by the time config reaches wandb it is a flattened, type-erased dict and the dataclass type needed to fill defaults is gone.
  - Used `dataclasses.asdict()` because it is the pattern already used elsewhere in this module's tests and round-trips safely under `dacite(strict=True)`.
  - Wrote the failing test first, per the repo's "add a failing test first" guideline.

---

## Pull Request

**PR Link:** https://github.com/ai2cm/ace/pull/1350

**PR Description:**

When a run is logged to Weights & Biases, only the config values explicitly set in the yaml file were captured — default field values defined on the `ModuleConfig` dataclass were silently omitted, because `ModuleSelector` kept the raw yaml dict in its `config` field and never serialized the defaults Python fills in when the `ModuleConfig` is instantiated. This PR normalizes `ModuleSelector.config` to the built instance's fully-defaulted values so defaults are captured wherever the config is serialized.

Changes:

- `fme.core.registry.module.ModuleSelector.__post_init__`: set `self.config = dataclasses.asdict(self._instance)` after building the instance.
- `fme.core.registry.test_module_registry`: add a mock builder with a default field and a test asserting the default appears in the serialized config.

- [x] Tests added
- [ ] If dependencies changed, "deps only" image rebuilt and `latest_deps_only_image.txt` updated

Resolves #596


**Maintainer Feedback:**
- Reviewed and approved by maintainer **mcgibbon** with "LGTM" — approved on the first pass with no requested changes.
- Maintainer enabled auto-merge (squash) and the PR merged once CI completed. **7 checks passed** (ruff, ruff-format, mypy, and the test suites).

**Status:** Merged into `ai2cm:main` (squash-merged as commit `5ecc4d4`).

---

## Learnings & Reflections

### Technical Skills Gained

- **Reading a bug across abstraction layers.** Learned to trace data from the yaml config through dataclass instantiation, `dataclasses.asdict()` serialization, and finally into `wandb.init(config=...)` — and to identify which layer actually causes a symptom versus where the symptom is visible.
- **Dataclass internals.** Understood the difference between a dataclass *field* (serialized by `dataclasses.asdict()`) and a plain instance attribute like `self._instance` (ignored by it), and how `__post_init__` is the right place to normalize state.
- **The registry / plugin pattern.** Saw how `ModuleSelector` stores `type` + a raw `config` dict to defer choosing a concrete `ModuleConfig` subclass until runtime, and why that indirection is what dropped the defaults.
- **Serialization round-tripping.** Confirmed a change is safe by reasoning about load compatibility (`dacite(strict=True)` accepting a fuller dict) rather than just checking that the write looked right.
- **Professional git/PR workflow.** Failing-test-first (red → green), running `pre-commit` (ruff/ruff-format/mypy) before pushing, writing a clean commit message, amending a bad message with `git commit --amend -F`, and filling out a repo's PR template.

### Challenges Overcome

- **The fix was not where I first thought.** I initially believed the change belonged in `_configure_wandb` (the wandb-logging code). Investigation showed that by the time the config reaches there it is a flattened, type-erased dict — the dataclass type needed to fill defaults is already gone. The fix had to move up to `ModuleSelector`, the last place the type still exists.
- **Verifying without access to the reporting environment.** I could not open the maintainer's private wandb run, so I verified at the serialization layer instead — asserting the default appears in `dataclasses.asdict(selector)["config"]`, which is exactly what `wandb.init` receives.
- **Backwards-compatibility risk.** The repo flags checkpoint loading compatibility as critical. I had to reason through whether normalizing `config` could break loading, and confirm it via the frozen/latest backwards-compatibility tests.
- **A git mishap.** Accidentally saved a mangled one-line commit message in vim; recovered cleanly with `git commit --amend -F <file>` since nothing had been pushed yet.

### What I'd Do Differently Next Time

- **Trace the data flow before assuming a fix location.** Confirming where the type information is lost *first* would have saved me from anchoring on the wandb file.
- **Use a message file from the start** (`git commit -F`) for any multi-paragraph commit, to avoid terminal paste collapsing newlines.
- **Ask about scope earlier.** My fix normalizes `config` for all consumers, not only wandb; I would raise that tradeoff in the PR description proactively (which I did here) or in the issue thread before implementing.

---

## Resources Used

- GitHub issue [ai2cm/ace #596](https://github.com/ai2cm/ace/issues/596) and the maintainer's guidance in the thread.
- The repository's `AGENTS.md` / `CLAUDE.md` for conventions (testing with `python -m pytest`, `pre-commit`, branch naming, commit rules, checkpoint backwards-compatibility policy, PR template).
- Existing code as reference for idiomatic patterns: `fme/core/registry/module.py`, `fme/core/registry/registry.py`, `fme/ace/train/train.py`, and `fme/core/registry/test_module_registry.py` (which already used `dataclasses.asdict(selector.module_config)`).
- Python docs for [`dataclasses`](https://docs.python.org/3/library/dataclasses.html) (`asdict`, `__post_init__`, fields vs. attributes).
- The [`dacite`](https://github.com/konradhalas/dacite) library docs for `from_dict` / `strict` behavior.
- [Weights & Biases docs](https://docs.wandb.ai/) for how `wandb.init(config=...)` records run configuration.


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
