## TDD - Write Tests Then Code

> **MUST:** All development work MUST follow TDD. All code reviews MUST verify test coverage.

Tests are the primary tool for proactively discovering issues and preventing regressions.

Design everything from the perspective of a future investigator:

- Before writing code, ask: "What tests would catch regressions?"
- Before committing, ask: "Do my tests cover the edge cases?"

**Practical guidelines:**

- Unit tests document expected behavior
- Test edge cases explicitly - they cause production incidents
- When fixing bugs, first write a test that reproduces it
