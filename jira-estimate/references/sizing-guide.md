# Sizing Guide

## Scale

| SP  | What it means |
|-----|---------------|
| 0.5 | Trivial config change — single field, no code, no dependencies |
| 1   | Simple, well-understood change. One class or flow, no edge cases |
| 2   | Small but requires thought. A few files, clear acceptance criteria |
| 3   | Medium complexity. Multiple components, some unknowns, needs testing |
| 5   | Complex. Cross-object, multiple layers, significant testing surface |
| 8   | Very complex or poorly defined. Break it up if possible |

## Rules of thumb

- If you're unsure between two sizes, pick the higher one
- If a ticket has missing acceptance criteria, don't estimate below 3 — you don't know what done looks like
- Deployment complexity counts (e.g. field deletion, type change, permission set rollout)
- Testing effort counts — a simple Apex change with 10 test class dependencies is not a 1
