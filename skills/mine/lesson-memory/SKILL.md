---
name: lesson-memory
description: >-
   Lesson memory persists a correction from three supplied facts. Use when a user or workflow provides what went wrong, why it went wrong, and the correct approach. 
   Don't use to diagnose failures or infer lessons from code history."
argument-hint: "<what went wrong> | <why it went wrong> | <correct approach>"
metadata:
  author: Vinícius Bôscoa
  github: https://github.com/virb30
  repository: https://github.com/virb30/ia-resources/skills
  related-skills: lesson-learned
---

# Lesson Memory

Persist one supplied correction as project memory. Treat the caller's three facts as authoritative;
format and index them without diagnosing the event or gathering additional evidence.

## Inputs

Require exactly these semantic inputs:

- what went wrong;
- why it went wrong;
- the correct approach.

Accept prose, structured fields, or equivalent labels in any language. Ask only for a missing or
ambiguous input; preserve supplied meaning instead of expanding it with inferred causes or advice.

## Step 1: Validate the correction

1. Map the supplied content to the three required inputs.
2. Verify that each input is concrete and that the correct approach states an actionable
   replacement for the error.
3. Ask the caller for only the unresolved input when validation fails.

*Done when:* all three inputs are present, distinct, and concrete.

## Step 2: Draft the lesson

1. Resolve the project root with `git rev-parse --show-toplevel`.
2. Resolve `<memory-dir>` to `<project-root>/.memory/lessons-learned`.
3. Read `<skill-dir>/assets/lesson.template.md` in full and replace every placeholder using only
   the three validated inputs.
4. Derive a concise title that distinguishes the correction without adding new claims.
5. Derive a lowercase kebab-case slug from the title and target
   `<memory-dir>/<YYYY-MM-DD>-<slug>.md`.

*Done when:* every template placeholder is resolved from the supplied correction and the target
path is known.

## Step 3: Detect collisions

Before writing, inspect `<memory-dir>/README.md` and every existing lesson file in `<memory-dir>`.
Compare the draft against existing entries by:

- exact target path or title;
- substantially equivalent error and cause;
- substantially equivalent correct approach;
- an existing lesson that the new correction would contradict or replace.

When no collision exists, continue to Step 4. When a collision exists:

1. Show the matching lesson path, its index description when present, and a concise old-versus-new
   comparison.
2. Ask the user for the final decision: overwrite the matching lesson or abandon the registration.
3. On overwrite, retain the existing lesson's path, replace its contents with the completed draft,
   and update its index description.
4. On abandon, leave every memory file unchanged and report that the registration was cancelled.

Treat an ambiguous answer as unresolved and ask again. Never create a second file for a known
semantic duplicate.

*Done when:* either no collision exists, or the user has explicitly chosen overwrite or abandon.

## Step 4: Persist the lesson and index

1. Create `<memory-dir>` when it does not exist.
2. Write one lesson file from the completed template.
3. Create `<memory-dir>/README.md` when absent, with the heading `# Lessons Learned Index`.
4. Add or replace exactly one bullet in the index:
   `- [<lesson title>](<lesson filename>) — <one-sentence description>`
5. Keep index entries sorted newest date first, then title alphabetically for equal dates.
6. Report the lesson path and index path.

*Done when:* exactly one lesson file contains the validated correction, exactly one index entry
links to it with a brief description, and every index link resolves to an existing file.

## Failure handling

- If the repository root cannot be resolved, ask the user for the project root before drafting.
- If the memory path is unwritable, report the resolved path and stop without creating partial
  memory.
- If an existing index is malformed, preserve it and ask before restructuring it.
