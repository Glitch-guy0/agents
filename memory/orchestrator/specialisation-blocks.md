# Specialisation Blocks

This file serves as the single source of truth for domain-specific constraints, contexts, and rules.
Core agents have no memory of these domains natively and rely on these blocks when invoked with a tag (e.g., `@android`, `@blockchain`).

## android
- minSdk 26
- use Jetpack Compose UI
- avoid long blocking tasks on the main thread
- do not store secrets in plaintext

## embedded
- prioritize memory efficiency
- minimize external dependencies
- avoid dynamic memory allocation where possible