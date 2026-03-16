---
name: no-separate-design-docs
description: Don't create separate Linear design documents — put design context in project description and requirements in ticket descriptions
type: feedback
---

Don't create separate Linear design documents. Put project-level design context (architecture, scope decisions, success criteria, failure modes) in the project description. Put ticket-level requirements (scope, acceptance criteria, testing) directly in ticket descriptions.

**Why:** Splitting information across a design doc + project + tickets means context is in multiple places and harder to find. Keep it consolidated — project description is the design doc.

**How to apply:** When creating Linear projects with multiple tickets, the project description should contain the architecture/design overview, and each ticket should be self-contained with all its requirements.
