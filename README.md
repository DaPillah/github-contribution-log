# Contribution #1: Add search filter/limit by date (e.g.: search X but show only conversations from last week/date/day/etc)

**Contribution Number:** 1  
**Student:** Justin  
**Issue:** [GitHub issue link](https://github.com/BasedHardware/omi/issues/4457)  
**Status:** Phase II [In Progress]

---

## Why I Chose This Issue

I chose this issue because it is a well-scoped, backend-focused feature request that aligns with my existing experience working with filters on logs and data. Since the fix lives in the Python backend rather than the mobile app, it felt like an approachable entry point into a real-world codebase without being overwhelmed by unfamiliar mobile development tools.

As a first-time open source contributor through CodePath, I wanted to work on something that would help me learn how search and retrieval logic works in a production AI application. This issue gives me the opportunity to understand how Omi stores and queries conversations, how API parameters get passed through a FastAPI backend, and how vector databases like Pinecone are used in practice.

---

## Understanding the Issue

### Problem Description

There is currently no way to filter conversation search results by date. Users who remember roughly when something happened but not the exact details have no way to narrow their search to a specific time window.

### Expected Behavior

Users should be able to search their conversations with an optional date filter (for example, "show only results from last week" or within a specific date range) so results are limited to the relevant time period.

### Current Behavior

Search returns all matching conversations regardless of when they occurred, making it difficult to find something when the user only has a rough idea of the timeframe.

### Affected Components

backend/utils/llm.py — where the retrieval and search logic lives  
backend/routers/ — where the API endpoint would need to accept date range parameters  
Pinecone — the vector database that stores conversations and would need to filter by date metadata  

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
