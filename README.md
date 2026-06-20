# Contribution #1: Add search filter/limit by date (e.g.: search X but show only conversations from last week/date/day/etc)

**Contribution Number:** 1  
**Student:** Justin  
**Issue:** [GitHub issue link](https://github.com/astronomer/astronomer-cosmos/issues/2822)  
**Status:** Phase II [In Progress]

---

## Why I Chose This Issue

I chose this issue because it involves improving error handling in a real production Python codebase used by data engineers working with dbt and Apache Airflow. As someone interested in data engineering and AI/ML workflows, working on Astronomer's Cosmos project gives me direct exposure to how production data pipelines handle failures and surface useful debugging information.

As a first-time open source contributor through CodePath, I wanted to work on something that would teach me meaningful Python patterns. This issue specifically helps me learn how subprocess output is captured and surfaced in exception messages (a skill directly applicable to any backend Python work).

---

## Understanding the Issue

### Problem Description

When Cosmos runs a dbt command via LoadMode.DBT_LS and it fails, the error message is often opaque (especially when both stdout and stderr appear empty) making it very difficult to diagnose why the command failed.

### Expected Behavior

When a dbt command fails, CosmosLoadDbtException should include the captured stdout, stderr, and process exit code in its message (even explicitly stating when no output was captured) so developers can quickly understand what went wrong.

### Current Behavior

The exception only shows Unable to run [... dbt ls ...] with little or no detail, leaving developers without enough information to debug the failure.

### Affected Components

cosmos/dbt/graph.py — specifically the run_command  
run_command_with_subprocess - functions where the exception is raised 

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

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
