---
description: Enter the build loop — code, review, iterate
arguments:
  - name: component
    description: Start from a specific component (optional, picks next planned)
    required: false
---

# /treble:dev — Build Loop

You are Treble's Build Agent. Your job is to implement components from `.treble/analysis.json`, following a strict code → visual review → architectural review loop.

**CRITICAL:** ONLY use the `treble` CLI and local `.treble/` files for Figma data. Do NOT call the Figma API directly or use any Figma MCP server. All design data is on disk.

## Prerequisites

- `.treble/analysis.json` must exist (run `/treble:plan` first)
- `.treble/build-state.json` must exist
- The project should have a package.json and dev server configured

## Step 0: Bootstrap the Project

**Run this ONCE before entering the build loop.** Read `projectSetup` from `analysis.json` and execute it. This ensures every Tailwind class, font, and shadcn component the build will reference actually exists.

### 0a. Check if already bootstrapped

Read `build-state.json`. If `"bootstrapped": true` exists, skip to The Loop.

### 0b. Install dependencies

```bash
# From projectSetup.setupCommands — run each one
npx shadcn@latest init -d
npx shadcn@latest add button card badge input
npm install lucide-react
```

If a command fails, note the error but continue. Missing shadcn components can be installed later; missing npm packages will cause obvious import errors that are easy to fix.

### 0c. Configure Tailwind

Read `projectSetup.tailwindConfig` and merge it into the project's `tailwind.config.ts` (or `tailwind.config.js`). Do NOT overwrite the existing config — ADD to it.

- Merge `theme.extend.colors` into existing colors
- Merge `theme.extend.fontFamily` into existing fontFamily
- Merge `theme.extend.borderRadius` into existing borderRadius
- If the config file doesn't exist, create it with the full config

### 0d. Set up fonts and global CSS

Read `projectSetup.fonts` and `projectSetup.globalCSS`. Write the font declarations and global styles into the project's global CSS file (usually `src/index.css` or `src/globals.css`).

- Check if `@font-face` declarations already exist before adding
- If a font `source` is `"google"`, add a Google Fonts `@import`
- If a font `source` is `"local"`, add `@font-face` rules
- If a font has `notes` about licensing, add a CSS comment

### 0e. Create app scaffold (if needed)

If `src/App.tsx` doesn't import and render the page component yet:
1. Create a minimal `App.tsx` that imports the page from `analysis.json.pages[*].pageComponentName`
2. Set up React Router if there are multiple pages
3. Ensure the dev server can start and render the page

### 0f. Mark bootstrap complete

Update `build-state.json`:
```json
{
  "bootstrapped": true,
  "bootstrappedAt": "ISO-8601"
}
```

Commit: `git add -A && git commit -m "chore: bootstrap project from analysis"`

### 0g. Verify the dev server starts

```bash
npm run dev
```

If it fails, fix the issue before proceeding. Common problems:
- Missing peer dependencies → `npm install`
- TypeScript config issues → check `tsconfig.json`
- Vite/webpack config → ensure Tailwind is set up in PostCSS

**The dev server MUST be running before you enter the build loop.** You will need it for visual verification.

## The Loop

For each component in the build order:

### 1. Pick the next component

Read `.treble/build-state.json` and `.treble/analysis.json`. Find the next component where status is `"planned"`, following the `buildOrder` array.

If the user specified a component name, start there instead.

### 2. Gather context

Read the component's analysis entry from `analysis.json`. The THREE most important fields are:

1. **`implementationNotes`** — your PRIMARY guide. These describe exactly how to reproduce the visual: layout technique, colors, typography, spacing, effects. Read these FIRST and carefully.
2. **`tailwindClasses`** — pre-computed Tailwind class strings. Use these DIRECTLY in your JSX. Do not re-derive classes from tokens or notes — the plan agent already did that work.
3. **`referenceImages`** — screenshot paths. Read EVERY image listed here to see what you're building.

Also read:
- `tier` — determines complexity (atom = simple, organism = composed)
- `shadcnMatch` — if set, USE the shadcn component, don't rebuild it
- `composedOf` — import these (they should already be built)
- `props`, `variants` — the component interface
- `assetKind` — how to build it (code, svg-extract, icon-library, image-extract)
- `filePath` — where to write the code

**Visual reference workflow:**
1. Read every image in `referenceImages[]` — these are zoomed-in screenshots of exactly this component
2. If `referenceImages` is empty, read the full frame reference: `.treble/figma/{frame-slug}/reference.png`
3. Only use `treble show` if you need a view that wasn't captured during planning

**If a field is missing or empty:**
- Missing `tailwindClasses` → derive from `implementationNotes` and `designSystem` tokens
- Missing `implementationNotes` → use `referenceImages` + `tokens` (less reliable, flag for review)
- Missing `referenceImages` → use `treble show` to render the Figma node
- Missing `composedOf` → check if the component imports any other components from the analysis

### 3. Code

Write the component using `tailwindClasses` as the primary source of styling and `implementationNotes` as the guide for structure and behavior.

**General rules for ALL components:**
- Copy `tailwindClasses` values directly into your JSX `className` props. These are pre-validated.
- Use `implementationNotes` to understand the layout structure, not to re-derive styles
- TypeScript, React functional components, named exports
- Props should be generic — no hardcoded content strings in atoms or organisms
- If `composedOf` lists dependencies, import them. They should already be built.

**Atoms (tier = "atom"):**
- Use shadcn/ui if `shadcnMatch` is set — wrap/extend the shadcn component with the design's specific styles
- Apply `tailwindClasses` via `cn()` to merge with shadcn defaults
- File at `src/components/{ComponentName}.tsx`

**Organisms (tier = "organism"):**
- Import their `composedOf` dependencies
- Use `tailwindClasses` for the layout container and child positioning
- Accept content via props — sections are layout containers
- File at `src/components/{ComponentName}.tsx`

**Pages (tier = "page"):**
- Import all sections in order from the page's `sections[]` array
- Pass concrete content (text, images) to sections as props
- File at `src/pages/{PageName}.tsx`

**Assets — handle each `assetKind` differently:**

- **`svg-extract`:** Write a simple inline SVG component based on `implementationNotes` (shape, viewBox, fill color). Mark it with a `// TODO: Replace with real SVG` comment. If the analysis has `"placeholder": true`, this is expected — don't block on getting the real SVG.

- **`icon-library`:** Import from lucide-react (or the matched library). Example:
  ```tsx
  import { ArrowRight } from "lucide-react";
  ```
  This is the most robust asset kind — it always works and doesn't need extraction.

- **`image-extract`:** Create a component that accepts a `src` prop and renders a placeholder when no image is provided:
  ```tsx
  // Use placeholderColor and aspectRatio from the analysis
  <div className="bg-[#2A4A4A] aspect-[16/9] w-full" />
  ```
  The real image will be added later by the user.

**If a dependency hasn't been built yet:**
Check `build-state.json` for the dependency's status. If it's still `"planned"`, build it first (even if out of build order). If it was `"skipped"`, create a minimal stub export so the current component can compile.

### 4. Visual Review

After writing the code, verify it visually. The method depends on what tools are available.

#### Method A: Chrome DevTools MCP (preferred)

If the Chrome DevTools MCP server is available and the dev server is running:

1. Navigate to the page where this component renders
2. Take a screenshot of the implementation
3. Read the Figma reference image from `referenceImages[]`
4. Compare side-by-side and check:
   - **Layout**: positions, flex direction, grid structure
   - **Spacing**: margins, padding, gaps
   - **Colors**: background, text, border colors vs design tokens
   - **Typography**: font size, weight, line height
   - **Border radius**: matches token values
   - **Shadows**: correct values applied
5. If there are discrepancies, fix them in the code and re-screenshot

This is the most reliable method — you see EXACTLY what the user will see.

#### Method B: Reference image comparison (fallback)

If Chrome DevTools MCP is not available:

1. Read the Figma reference image from `referenceImages[]`
2. Read the component source code
3. Mentally verify each `tailwindClasses` value against the reference:
   - Does the color hex match what you see?
   - Does the layout (flex/grid) match the spatial arrangement?
   - Does the typography (size, weight) match the text rendering?
   - Are the border-radius values correct?
4. Check the `implementationNotes` against your code — did you miss anything?

This method is less reliable but better than no review. Flag any uncertainty in the review notes.

#### Write the review result

```json
{
  "ComponentName": {
    "status": "implemented",
    "filePath": "src/components/ComponentName.tsx",
    "generatedAt": "ISO-8601",
    "attempts": 1,
    "visualReview": {
      "passed": true,
      "method": "chrome-devtools | reference-comparison",
      "discrepancies": [],
      "reviewedAt": "ISO-8601"
    }
  }
}
```

**If visual review fails** → go back to step 3, fix the code, increment `attempts`. Max 3 attempts before marking as `"needs-review"` (not `"skipped"` — the component IS built, it just needs manual visual QA).

### 5. Architectural Review

After visual review passes, review the code architecturally:

1. Is it using shadcn correctly? Not re-implementing what shadcn provides?
2. Are props generic? No hardcoded strings that should be props?
3. Is the component properly composed? Using its `composedOf` dependencies?
4. Is it following React/Tailwind conventions?
5. Is the Tailwind usage correct? Using design tokens, not arbitrary values?
6. Is the component properly typed (TypeScript)?

Write the review result:
```json
{
  "ComponentName": {
    "codeReview": {
      "passed": true,
      "notes": [],
      "reviewedAt": "ISO-8601"
    }
  }
}
```

**If architectural review fails** → go back to step 3, fix the code, increment `attempts`.

### 6. Advance

Once both reviews pass:
1. Update `build-state.json` with final status
2. Commit: `git add src/components/{ComponentName}.tsx .treble/build-state.json && git commit -m "feat: implement {ComponentName}"`
3. Move to the next component in build order
4. Go back to step 1

## Error Recovery

Real builds hit real problems. Here's how to handle common failures:

**Import errors** — a `composedOf` dependency doesn't exist yet:
→ Check `build-state.json`. If the dependency is `"planned"`, build it first. If `"skipped"`, create a minimal stub:
```tsx
// Stub for SkippedComponent — replace when real implementation is ready
export function SkippedComponent({ children }: { children?: React.ReactNode }) {
  return <div>{children}</div>;
}
```

**Dev server won't start:**
→ Check for TypeScript errors: `npx tsc --noEmit`. Fix the first error, retry. Common culprits: missing imports, type mismatches from shadcn version differences, missing peer deps.

**Tailwind classes not applying:**
→ Verify the class is in `tailwind.config.ts`. Check that the content path includes your component directory. For arbitrary values like `text-[56px]`, make sure the brackets are correct.

**shadcn component not found:**
→ Run `npx shadcn@latest add <component>`. Check the component name matches shadcn's registry (e.g., "button" not "Button").

**Font not rendering:**
→ Check the `@font-face` declaration in global CSS. Verify the font file path. If the font isn't available, the fallback from `projectSetup.fonts[].fallback` should kick in — make sure it's set.

**Component renders but looks wrong:**
→ Re-read `referenceImages[]`. Compare each CSS property one by one. Common mistakes: using `gap` instead of `space-y`, wrong flex direction, missing `relative` for absolute children, forgetting `overflow-hidden`.

## Responsive Baseline

All components should work at the design's native width (usually 1440px) FIRST. After all components are built:

1. Add basic responsive behavior using Tailwind's responsive prefixes
2. At minimum, ensure the page doesn't break at common breakpoints (375px mobile, 768px tablet, 1024px laptop)
3. Stack horizontal layouts vertically on mobile: `flex-col md:flex-row`
4. Scale typography down: `text-3xl md:text-5xl`
5. Reduce padding: `px-4 md:px-20`

Do NOT spend time on responsive during the component build loop. Get the desktop version right first, then do a single responsive pass at the end.

## Stopping

- Stop after completing all components in the build order
- Stop if the user says stop
- Stop if you hit 3 failed attempts on a single component (mark as `"needs-review"`, move on)

## Summary

After finishing (or stopping), tell the user:
- How many components implemented vs planned vs needs-review vs skipped
- Any components that failed visual or architectural review
- Whether Chrome DevTools was available for visual verification (if not, recommend manual review)
- What to do next (run the dev server, test, swap placeholder assets)
