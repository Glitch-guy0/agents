---
name: project-planner
description: >
  A multi-agent project planning orchestrator. Use this skill whenever the user says things like
  "plan this project", "I have an idea for...", "new plan", "help me plan", "plan for this project",
  or describes a project/product/system they want to build. Triggers on new project ideas, vague
  feature requests, or anything that needs structured decomposition into milestones and tasks.
  Also triggers if the user says "resume my project plan" or "continue planning". Even if the
  request seems simple, if it involves building or implementing something non-trivial, use this skill.
---

# Project Planner — Orchestrator

A living project planning system. Claude acts as an orchestrator, delegating to specialized internal
agents and maintaining a file structure that evolves with the project.

---

## Session Start Protocol

Before doing anything else:

1. Check if `[project-name]-plan/` already exists in the user's working directory.
   - If it **exists**: Tell the user — *"A plan for '[name]' already exists at `[name]-plan/`. Would you like to continue where you left off, or start fresh?"*
   - If it **doesn't exist**: Create the folder `[inferred-name]-plan/` and initialize empty files (see File Structure below).
   - If project name is unclear: ask the user before creating anything.

2. If resuming: read all existing files in the `-plan/` folder to reconstruct full project state before responding.

---

## Internal Agents

Read the relevant agent file **only when you need it**. Do not load all agents at once.

| Agent | File | When to load |
|---|---|---|
| Technical Docs | `agents/technical-docs.md` | Step 1 — parsing requirements |
| Milestone Manager | `agents/milestone-manager.md` | Steps 1, 2, 7, 8, 9 |
| Implementation Manager | `agents/implementation-manager.md` | Steps 4, 9 |

---

## The Planning Loop

### Step 1 — Parse & Decompose
When a new request or requirements dump arrives:
- Load `agents/technical-docs.md` → update schema, UML, class, system design, and technical docs files
- Load `agents/milestone-manager.md` → update milestones file
- Always apply **diff color coding** (see Color Rules below) when updating any file
- Proceed to Step 2

### Step 2 — Milestone Readiness Check
After milestones are updated, check: does `milestones.md` have at least one milestone with status `READY`?
- **Yes** → Step 3
- **No** → Step 5

### Step 3 — Implementation Offer
Present the user with:
- A plain-language summary of what is ready to implement
- How much of the overall plan this covers (e.g., "This covers ~30% of the planned scope")
- Ask: *"Ready to generate the implementation plan for this, or do you want to refine requirements first?"*
  - **Yes, implement** → Step 4
  - **Refine first** → Step 5

### Step 4 — Implementation Plan
Load `agents/implementation-manager.md`.
- Generate detailed task list
- Update `implementation.md` with state diagram + task blocks
- Apply diff color coding
- Return to Step 2 to check remaining milestones

### Step 5 — Requirements Gap
Ask the user:
*"Do you have more requirements in mind, or should I ask you questions to help fill the gaps?"*
- **Requirements dump** → Step 6
- **Interview me** → Step 7

### Step 6 — Requirements Dump
Accept the user's input as-is. Loop back to **Step 1**.

### Step 7 — Interview Mode
Load `agents/milestone-manager.md`.
- Identify the first incomplete or vague milestone
- Ask the user **one question at a time** to fill gaps
- Each question should reference what's already known and why the gap matters
- After each answer: update `milestones.md`, apply diff color coding, proceed to next gap
- When a milestone is fully defined → Step 8

### Step 8 — Milestone Complete
Ask the user:
*"[Milestone name] is now fully defined. Want to implement it now, or continue defining the next milestone?"*
- **Implement** → Step 4
- **Continue defining** → Step 7 for the next milestone

### Step 9 — Completion Check
After each implementation or milestone update:
- Are there unfinished milestones with gaps? → Step 7
- Are there ready milestones not yet implemented? → Step 2
- Are all milestones defined AND implemented? → **Step 10**

### Step 10 — Done
Tell the user the plan is complete. Summarize:
- Total milestones
- Total tasks generated
- Files created
- Suggest next steps (e.g., hand off tasks to a developer, export docs)

---

## File Structure

All files live in `[project-name]-plan/`:

| File | Diagram Type | Managed By |
|---|---|---|
| `schema.md` | Mermaid `erDiagram` | Technical Docs |
| `uml.md` | Mermaid `zenuml` (fallback: `sequenceDiagram`) | Technical Docs |
| `classes.md` | Mermaid `classDiagram` | Technical Docs |
| `technical-docs.md` | Mermaid `mindmap` | Technical Docs |
| `system-design.md` | Mermaid `graph TD` or `C4Context` | Technical Docs |
| `milestones.md` | Mermaid `requirementDiagram` | Milestone Manager |
| `implementation.md` | Mermaid `stateDiagram-v2` + task blocks | Implementation Manager |

Each file is **fully regenerated** on update (not appended). History is preserved through color coding.

---

## Color Rules (Diff Coding)

Apply on every file update using Mermaid `classDef` and `style` directives:

```
NEW elements:      fill:#FF6B35,color:#fff,stroke:#FF6B35       (bright orange)
MODIFIED elements: fill:#FFD166,color:#333,stroke:#FFD166       (amber yellow)
EXISTING elements: fill:#4A4A5A,color:#ccc,stroke:#6C757D       (muted grey)
REMOVED elements:  fill:#DC3545,color:#fff,stroke:#DC3545       (red, kept for one revision)
```

**Frame separation rule**: Where a file contains multiple logical sections (e.g., multiple milestones),
wrap each section with a Mermaid `subgraph` labelled with the section name. New subgraphs get the NEW color on their label node.

After two revisions without changes, an element graduates to EXISTING styling.

---

## General Rules

- Never ask more than one question at a time during interview mode
- Always show a plain-language summary alongside any diagram update
- Never overwrite an existing `-plan/` folder without user confirmation
- If a diagram type isn't supported by the user's renderer, fall back gracefully and note it
- All task descriptions in `implementation.md` must be self-contained (see Implementation Manager)
- Keep orchestration state in your context — do not rely on the user to track which step you're on
