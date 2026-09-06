[tooling_verdict.md](https://github.com/user-attachments/files/31883564/tooling_verdict.md)
# Verdict: Use PyJobShop

| Requirement | PyJobShop support | Evidence |
|---|---|---|
| Optional task selection | `add_task(..., optional=True)` | Used directly in toy example |
| Sequence-dependent setup times | `Model.add_setup_time(machine, task1, task2, duration)` | Used directly (Section A travel times) |
| Breaks (corridor block window) | `Machine.breaks = [(start, end), ...]` | Used directly (window 60–240 min, blocked outside it) |
| Multiple modes | `Model.add_mode(task, resource, duration, ...)` — a task can have >1 mode by calling `add_mode` more than once for different resources/durations | Present in API (not exercised with >1 mode per task in the toy run, but signature confirmed) |
| Release dates / deadlines / due dates | `add_task(earliest_start=..., latest_end=...)` | Used directly |

## Verdict

**Use PyJobShop.**
