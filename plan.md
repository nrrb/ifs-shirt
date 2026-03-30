# IFS Morphing Grid Tool — Spec

## Overview

A full-page Vue component that renders an N×N grid of IFS fractals, where the four corners are fixed to specific IFS systems and all interior points are bilinearly interpolated between them.

## IFS Systems

Each IFS system is defined as 4 affine functions. Each function has parameters `a, b, c, d, tx, ty, p` where the transformation is:

```
x' = a*x + b*y + tx
y' = c*x + d*y + ty
```

Use these exact corner systems and correspondence ordering (slot index must match across all four corners for the morph to be semantically coherent):

**Fern** (bottom-left, position 0,0):
- Slot 1: [0.00, 0.00, 0.00, 0.16, 0.00, 0.00, 0.01]
- Slot 2: [0.85, 0.04, -0.04, 0.85, 0.00, 1.60, 0.85]
- Slot 3: [0.20, -0.26, 0.23, 0.22, 0.00, 1.60, 0.07]
- Slot 4: [-0.15, 0.28, 0.26, 0.24, 0.00, 0.44, 0.07]

**Sierpinski** (bottom-right, position 1,0):
- Slot 1: [0.50, 0.00, 0.00, 0.50, 0.00, 0.00, 0.34]
- Slot 2: [0.50, 0.00, 0.00, 0.50, 0.25, 0.50, 0.33]
- Slot 3: [0.50, 0.00, 0.00, 0.50, 0.50, 0.00, 0.33]
- Slot 4: [0.00, 0.00, 0.00, 0.00, 0.50, 0.50, 0.00]

**Snowflake** (top-left, position 0,1):
- Slot 1: [0.333, 0.000, 0.000, 0.333, 0.000, 0.000, 0.25]
- Slot 2: [0.167, -0.289, 0.289, 0.167, 0.333, 0.000, 0.25]
- Slot 3: [0.167, 0.289, -0.289, 0.167, 0.500, 0.289, 0.25]
- Slot 4: [0.333, 0.000, 0.000, 0.333, 0.667, 0.000, 0.25]

**Dragon** (top-right, position 1,1):
- Slot 1: [0.00, -0.50,  0.50,  0.00,  0.00,  0.00, 0.25]
- Slot 2: [0.00,  0.50, -0.50,  0.00,  0.50,  0.50, 0.25]
- Slot 3: [0.00,  0.50, -0.50,  0.00,  1.00,  0.00, 0.25]
- Slot 4: [0.00, -0.50,  0.50,  0.00,  0.50,  0.00, 0.25]

## Interpolation

For grid position (i, j) where i and j each run from 0 to N-1, compute s = i/(N-1) and t = j/(N-1). Then bilinearly interpolate all parameters:

```
θ(s,t) = (1-s)(1-t)*θ_Fern + s(1-t)*θ_Sierpinski + (1-s)t*θ_Snowflake + st*θ_Dragon
```

After interpolating, renormalize the p values so they sum to 1.

All four coordinate spaces must be normalized into [0,1]² before interpolation. Compute the bounding box of each attractor by running a warmup iteration pass, then map to [0,1]² using that bounding box. This prevents morphed shapes from drifting off canvas.

## IFS Rendering

Use the chaos game algorithm: start at (0,0), on each iteration pick a function with probability proportional to its p value, apply it, and plot the resulting point. Skip the first 50 iterations as warmup. Render points onto an HTML canvas using ImageData for performance.

## Grid Layout

Render each cell as a canvas element in a CSS grid. Cells should be square. The full grid should fit within the viewport. Label the four corners with the system name. Labels appear **outside** the rendered canvas area (not overlaid on the fractal) and are **not included** in exported images.

## Controls Panel

Provide a controls panel with:
- **N** — integer slider from 5 to 20, default 5
- **Iterations** — slider from 1 to 10,000, default 5,000, displayed as a plain integer
- **Point color** — color picker, default white (`#ffffff`)
- **Background color** — color picker, default near-black (`#0a0a0f`)
- **Invert colors** — checkbox; when checked, renders black points on white background. In exported PNG, the background becomes transparent (alpha = 0) rather than white, leaving only the black point pixels.
- **Export size (inches)** — number input (1–60), default 12. Output resolution = 300 × inches pixels square (e.g. 12 in → 3600×3600). Shows a live pixel count preview below the input.
- **Re-render button** — reruns the chaos game with current settings
- **Export button** — see export spec below

## Export

When the user clicks Export, render the entire grid to a single offscreen canvas at 300 × exportInches pixels square. Embed 300 DPI metadata in the PNG using a pHYs chunk (2953 pixels per meter). Download the result as a PNG file named `ifs-grid.png`.

When "Invert colors" is active, the exported PNG uses a transparent background with black points (not a white background).

The pHYs PNG chunk must be injected manually into the PNG byte stream after the IHDR chunk since the Canvas API does not expose DPI metadata natively. Write a helper function that takes a base64 PNG from `canvas.toDataURL()`, parses the PNG bytes, inserts the pHYs chunk, and returns a new Blob for download.

## Override Corner

An "Override Corner" panel in the controls allows the user to replace any corner's IFS functions via text paste.

**UI:**
- Dropdown to select which corner: Fern (0,0), Sierpinski (1,0), Snowflake (0,1), Dragon (1,1)
- Textarea for pasting function definitions
- Apply button — parses and applies the override, then triggers a full re-render
- Reset button — restores the corner to its default parameters and clears the textarea
- Status line: green "✓ 4 functions loaded" on success, red "✗ \<error\>" on failure

Overridden corners show a `*` indicator in both the corner dropdown and the grid corner labels.

**Accepted paste formats:**

Slot list format:
```
- Slot 1: [0.50, 0.00, 0.00, 0.50, 0.00, 0.00, 0.25]
- Slot 2: [0.00, -0.50, 0.50, 0.00, 0.50, 0.00, 0.25]
- Slot 3: [0.00, 0.50, -0.50, 0.00, 0.50, 0.50, 0.25]
- Slot 4: [0.50, 0.00, 0.00, 0.50, 0.50, 0.50, 0.25]
```

Raw bracket format:
```
[0.50,  0.00,  0.00,  0.50,  0.00,  0.00, 0.25]
[0.00, -0.50,  0.50,  0.00,  0.50,  0.00, 0.25]
```

Bare numbers format:
```
0.50  0.00  0.00  0.50  0.00  0.00  0.25
0.00 -0.50  0.50  0.00  0.50  0.00  0.25
```

**Validation:**
- Exactly 4 functions required
- Each function must have exactly 7 numeric values
- All p values must be non-negative
- If p values don't sum to 1, warn but auto-renormalize before applying
- Blank lines and `#` comment lines are ignored

## Performance

Each cell renders independently. Render cells sequentially with `await new Promise(r => setTimeout(r, 0))` between each to keep the UI responsive. Show a progress indicator during rendering (e.g. "Rendering 3/16...").

## Vue Architecture

- Single component, Composition API with `<script setup>`
- Use `ref` and `computed` for reactive state
- Canvas elements accessed via template refs
- No external dependencies — use vanilla canvas API throughout
