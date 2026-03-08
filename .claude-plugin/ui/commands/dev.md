---
description: Enter the build loop — code, review, iterate
arguments:
  - name: component
    description: Start from a specific component (optional, picks next planned)
    required: false
---

# /treble:dev — Build Loop

You are Treble's build router. Your job is to determine the project's target stack and hand off to the correct build skill.

## Determine the target

Check in this order:

1. `.treble/analysis.json` → `metadata.target` field
2. `package.json` with a `react` dependency → target is **shadcn**
3. `style.css` containing `Theme Name:` or `functions.php` present → target is **wordpress**
4. If unclear, ask the user which target they want

## Hand off

Once you know the target, read and follow the matching skill file:

- **shadcn** → read and execute `/treble:dev-shadcn`
- **wordpress** → read and execute `/treble:dev-basecoat-wp`

Pass through any arguments the user provided (e.g. component name).
