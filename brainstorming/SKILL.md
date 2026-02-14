---
name: brainstorming
version: 0.1.0
description: "Turn vague ideas into a validated design/spec through structured brainstorming. Use before any creative work (new features, UI/components, behavior changes, refactors) and whenever a user asks to brainstorm, define requirements, propose approaches, or write a design doc."
---

# Brainstorming Ideas Into Designs

## Overview

Help turn ideas into fully formed designs and specs through natural collaborative dialogue.

Start by understanding the current project context, then ask questions one at a time to refine the idea. Once you understand what you're building, present the design in small sections (200–300 words), checking after each section whether it looks right so far.

## The Process

**Understanding the idea:**
- Review current project context first (key files/docs, recent commits, existing patterns).
- Ask one question per message to refine the idea (prefer multiple-choice when possible).
- Focus on: purpose, constraints, success criteria, non-goals.

**Exploring approaches:**
- Propose 2–3 approaches with trade-offs.
- Recommend one approach and explain why.

**Presenting the design:**
- Present the design in 200–300 word chunks and confirm after each chunk.
- Cover: architecture, components, data flow, error handling, testing.
- Be ready to backtrack if the user says something doesn’t fit.

## After the Design

**Documentation:**
- Write the validated design to `docs/plans/YYYY-MM-DD-<topic>-design.md` when that convention exists; otherwise create `docs/plans/` or ask where to put it.
- If you have a writing-quality skill available, use it before finalizing the doc.
- If the repo is under git and the user wants it tracked, commit the design doc.

**Implementation (if continuing):**
- Ask: “Ready to set up for implementation?”
- If the user wants a separate workspace, use a git worktree.
- Create a concrete implementation plan before writing code.

## Key Principles

- **One question at a time** — Don’t overwhelm with a questionnaire.
- **Multiple choice preferred** — Faster to answer and reduces ambiguity.
- **YAGNI ruthlessly** — Remove unnecessary scope/features from designs.
- **Explore alternatives** — Always consider 2–3 viable approaches.
- **Incremental validation** — Confirm each design chunk before continuing.
- **Be flexible** — Adjust quickly when something doesn’t fit.
