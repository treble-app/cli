---
description: Compare implementation screenshot against Figma reference
arguments:
  - name: component
    description: Component or page name to compare (e.g. "HeroSection", "Homepage")
    required: true
---

# /treble:compare — Visual Comparison

Compare a built component's rendered output against the Figma reference image. This does a REAL side-by-side comparison, not just a "does it render" check.

## Steps

### 1. Find the Figma reference

Look up the component in `.treble/analysis.json`:
- Find `referenceImages` paths — these are the Figma screenshots on disk
- If no referenceImages, render one: `treble show "{nodeId}" --frame "{frameName}" --json`

### 2. Screenshot the implementation (via chrome-devtools-tester subagent)

Spawn a `chrome-devtools-tester` subagent:

```
Navigate to the running dev server (check localhost:3000, 3001, 5173, or whatever port is configured).
Set viewport to 1440px width.
Wait for full page load (network idle).
Take a full-page screenshot and save to .treble/screenshots/{component}-impl.png

If the component is a section (not a full page), scroll to it and take a targeted screenshot.

Return the screenshot file path.
```

### 3. Compare (via general-purpose subagent)

Spawn a `general-purpose` subagent that reads BOTH images:

```
You are a pixel-perfectionist UI reviewer. Compare these two images:

FIGMA DESIGN: Read {figma reference path}
IMPLEMENTATION: Read .treble/screenshots/{component}-impl.png

Go section by section. For EACH area, check:
- LAYOUT: element positions, flex direction, grid structure, alignment
- SPACING: margins, padding, gaps between elements
- COLORS: backgrounds, text colors, borders, gradients
- TYPOGRAPHY: font size, weight, line height, letter spacing, family
- SHAPES: border radius, shadows, decorative elements
- CONTENT: is placeholder content roughly appropriate?

Be BRUTAL. Flag every difference no matter how small. This is about pixel perfection.

Rate each section: MATCH / CLOSE / WRONG

Return:
{
  "overall": "MATCH|CLOSE|WRONG",
  "sections": [
    {
      "name": "section name",
      "rating": "MATCH|CLOSE|WRONG",
      "discrepancies": ["specific issue"],
      "fix": "specific code change"
    }
  ],
  "summary": "one sentence overall assessment"
}
```

### 4. Report and fix

Show the user the comparison results. If discrepancies found:
1. Fix the implementation code
2. Re-compare (max 2 fix-compare cycles)
3. Update `.treble/build-state.json` with the review result

**IMPORTANT:** The subagent approach keeps images out of the main context. NEVER read PNG files directly in the main conversation.
