# PR Timeline Strategy (5 Days)

## Overview

This document outlines the PR creation and merge schedule for introducing subtle bugs into the todo-app-api over 5 days.

---

## Day 1 - Cleanup & Foundation

| Time | Action | PR Title | Bug Introduced |
|------|--------|----------|----------------|
| 9:00 AM | Create PR | "Chore: Remove deprecated endpoints" | Cleanup |
| 11:00 AM | Merge | ↑ | |
| 2:00 PM | Create PR | "Refactor: Extract rate limiter" | None (setup) |
| 4:00 PM | Merge | ↑ | |

---

## Day 2 - Validation & Correlation IDs

| Time | Action | PR Title | Bug Introduced |
|------|--------|----------|----------------|
| 9:00 AM | Create PR | "feat: Add input validation" | Schema gap |
| 11:00 AM | Add comment | "Should we use transform:true?" | Hint |
| 2:00 PM | Merge | ↑ | |
| 3:00 PM | Create PR | "Enhancement: Add correlation IDs" | Works correctly |
| 5:00 PM | Merge | ↑ | |

---

## Day 3 - Logging & Caching

| Time | Action | PR Title | Bug Introduced |
|------|--------|----------|----------------|
| 9:00 AM | Create PR | "Refactor: Add service logging" | Breaks correlation |
| 10:00 AM | Add comment | "LGTM, good separation of concerns" | Misleading |
| 12:00 PM | Merge | ↑ | |
| 2:00 PM | Create PR | "Performance: Add request caching" | Cache TTL bug |
| 3:00 PM | Add comment | "Tested 30min, stable memory" | Misleading |
| 5:00 PM | Merge | ↑ | |

---

## Day 4 - Timeouts & Configuration

| Time | Action | PR Title | Bug Introduced |
|------|--------|----------|----------------|
| 9:00 AM | Create PR | "Fix: Cache cleanup timing" | Closure bug |
| 11:00 AM | Merge | ↑ | |
| 1:00 PM | Create PR | "Configure axios timeout" | Sets 5s |
| 2:00 PM | Merge | ↑ | |
| 3:00 PM | Create PR | "Performance: Fail-fast timeout" | Sets 3s mismatch |
| 4:00 PM | Add comment | "Matches axios config" | Incorrect hint |
| 5:00 PM | Merge | ↑ | |

---

## Day 5 - Retry & Race Conditions

| Time | Action | PR Title | Bug Introduced |
|------|--------|----------|----------------|
| 9:00 AM | Create PR | "Reliability: Add retry mechanism" | Aggressive retries |
| 10:00 AM | Add comment | "Exponential backoff is best practice" | Misleading |
| 12:00 PM | Merge | ↑ | |
| 2:00 PM | Create PR | "Optimize rate limiter checks" | Race condition |
| 3:00 PM | Add comment | "Nice optimization!" | Misleading |
| 4:00 PM | Merge | ↑ | |
| 5:00 PM | Create PR | "docs: Add architecture docs" | Red herrings |
| 5:30 PM | Merge | ↑ | **Complete** |

---

## Summary of Bugs

| Bug | Introduced In | Description |
|-----|---------------|-------------|
| Schema validation gap | Day 2 - PR #3 | `transform: false` doesn't coerce string to number |
| Correlation ID loss | Day 3 - PR #5 | Service creates own logger, loses correlation |
| Cache TTL bug | Day 3 - PR #6 | `<=` instead of `<` keeps entries one cycle too long |
| Closure capture bug | Day 4 - PR #7 | Cleanup interval references old Map after recreation |
| Timeout mismatch | Day 4 - PR #9 | Service timeout (3s) vs axios timeout (5s) |
| Aggressive retry | Day 5 - PR #10 | 1.5x backoff (should be 2x), small jitter |
| Rate limiter race | Day 5 - PR #11 | TOCTOU vulnerability in check-then-increment |
