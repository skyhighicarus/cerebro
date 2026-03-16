# Development Guidelines

This is a living document for a learning-focused workspace. Projects here span Python, Rust, and beyond — most are exploratory. The goals are to finish what we start, internalize core principles, and build a referenceable body of work over time.

---

## Meta - Maintaining This Document

### Writing Effective Guidelines

When adding new rules to this document, follow these principles:

**Core Principles (Always Apply):**
1. Use absolute directives for behavioral rules (NEVER/ALWAYS). Use imperative verbs for principles.
2. Lead with why - Explain the problem before the solution (1-3 bullets max)
3. Be concrete - Include actual commands/code when applicable
4. Minimize examples - One clear point per code block
5. Bullets over paragraphs - Keep explanations concise

**Optional Enhancements (Use Strategically):**
- X/√ examples: Only when the antipattern is subtle
- "Warning Signs" section: Only for gradual mistakes
- "General Principle": Only when abstraction is non-obvious

**Anti-Bloat Rules:**
- Don't add "Warning Signs" to obvious rules
- Don't show bad examples for trivial mistakes
- Don't write paragraphs explaining what bullets can convey

---

## Learning and Retention

- **Explain the logic from first principles while implementing** — don't just write correct code, narrate *why* it works so the reasoning sticks
- When revisiting a concept (e.g., linear regression, ownership in Rust), briefly recap the mental model before diving into syntax
- Prefer building understanding over copy-paste solutions
- Flag when a pattern or concept is reappearing across projects — connecting the dots reinforces retention
- Finish projects before starting new ones; incomplete work decays faster than completed work

---

## Core Development Principles

These principles are listed in priority order. When they conflict, earlier principles take precedence.

### KISS (Keep It Simple, Stupid)
- **Straightforward solutions beat clever ones**
- Prefer readability over performance unless profiling proves otherwise
- Avoid over-engineering — if the simple version works, ship it

### YAGNI (You Aren't Gonna Need It)
- **Implement only what's needed NOW**
- No speculative features or abstractions
- Delete unused code aggressively — if it's not being called, remove it

### Single Responsibility Principle
- One class/module = one reason to change
- One function = one job
- Separate concerns: data access, business logic, presentation

### Open-Closed Principle
- Extend behavior without modifying existing code
- Add extension points (protocols/interfaces) only when multiple consumers exist today, not speculatively

---

## Decision Framework for New Code

Before writing code, ask in this order:

1. **YAGNI**: Do we need this NOW, or is this speculative?
2. **KISS**: What's the simplest solution that could work?
3. **SRP**: Does this function/class have exactly ONE responsibility?
4. **Open-Closed**: Can we extend this without modifying existing code?

If checks conflict, the earlier check wins. YAGNI and KISS override premature abstraction from SRP or Open-Closed.

