# Pixel Topology Rules

A browser-based tool for defining, testing, and matching binary pixel neighborhood patterns on rasterized images. Create small NxN topology rules that describe local pixel configurations (black, white, or "don't care"), then scan any binary image to find all positions where those patterns occur.

Built for analyzing rasterized typography, geometric primitives, and any domain where local pixel structure matters.

**[Live Demo](https://mansgreback.github.io/pixel-topology-rules/)**

---

## Overview

Pixel Topology Rules treats binary images as discrete grids where each pixel is either black (foreground) or white (background). You define small pattern templates — typically 3x3 to 9x9 — specifying which cells must be black, which must be white, and which can be anything. The tool then slides each pattern across the image and marks every position where the pattern matches.

This is conceptually similar to convolution kernels in image processing, but operating on discrete binary states rather than continuous values, and using exact logical matching rather than weighted sums.

### What it's useful for

- **Font engineering**: Detecting specific junction types, serif terminals, stroke endings, and corner configurations in rasterized glyphs
- **Pattern classification**: Finding and categorizing recurring local structures in binary images
- **Feature detection**: Identifying topological features like T-junctions, L-corners, endpoints, crossings, and curve inflection points at the pixel level
- **Quality analysis**: Spotting rendering artifacts, aliasing patterns, or structural anomalies in rasterized output
- **Education**: Understanding how local neighborhood operations work in binary image analysis

---

## Features

### Rule Editor

- **NxN grid** (2x2 to 12x12): Click cells to cycle through black / white / any
- **Sub-pixel anchor placement**: Right-click to place anchor nodes at cell center, edge midpoint, or corner — anchors define the "hit position" reported when a pattern matches
- **Multiple anchors per rule**: A single pattern can report hits at multiple anchor positions
- **Nudge controls**: Shift the entire pattern (cells + anchors) in any direction

### Symmetry Transforms (D4 Dihedral Group)

Each rule can optionally match under any combination of:

- **Negative** (color inversion): Also matches with black and white swapped
- **Rotational**: Matches at 0, 90, 180, and 270 degrees
- **Mirror**: Matches the horizontal reflection

With all three enabled, a single rule can match up to 16 orientations (4 rotations x 2 mirrors x 2 color states), with automatic deduplication of equivalent transforms.

### Exclusion Rules

Rules can be marked as **exclusion rules** — instead of adding match positions, they _remove_ positions found by other rules. This implements a two-pass detection pipeline:

1. All inclusion rules scan the image and collect hit positions
2. All exclusion rules scan the image and remove any overlapping positions

This allows precise control: define broad inclusion patterns, then carve out specific false-positive configurations.

### Detection & Visualization

- **Real-time scanning**: The image is scanned whenever rules change, with immediate visual feedback
- **Color-coded hits**: Green dots for the rule currently being edited, red dots for saved rules
- **Hover highlighting**: Hovering over a match position highlights which rules produced that hit
- **Selection-to-rule**: Drag a rectangle on the image to capture a region as a new rule template

### Image Support

- **7 built-in test images**: Typography samples (serif, sans-serif, punctuation, multi-size), geometric shapes, and mixed content
- **Drag-and-drop**: Drop any image onto the canvas to load it (auto-converted to binary, max 300px)
- **Automatic binarization**: RGB images are converted using luminance thresholding (ITU-R BT.601)

### Save / Load

Rules are saved as JSON files (`pixel_rules.json`), enabling:

- Persistent storage of rule sets
- Sharing rule configurations between users
- Programmatic rule generation and analysis

---

## Usage

### Quick Start

1. Open `index.html` in any modern browser (no server required)
2. Click cells in the grid editor to define a pattern (black/white/any)
3. Click **+ Save Rule** to add it to the rule set
4. Observe match positions highlighted on the test image
5. Switch between test images using the tabs above the canvas

### Creating a Rule

1. Set the grid size (e.g., 3x3 for simple corners, 5x5 for more specific patterns)
2. Click cells to cycle: gray (any) → black → white → gray
3. Right-click cells to place/remove anchor nodes (green dots with yellow outline)
4. Toggle symmetry options: Negative, Rotational, Mirror
5. Click **+ Save Rule**

### Editing a Rule

- Click any saved rule thumbnail in the sidebar to load it into the editor
- The editor shows "Done Editing" instead of "+ Save Rule" while editing
- Changes are applied to the image in real-time
- Click **Done Editing** to finish

### Selection to Rule

1. Click and drag on the image to select a rectangular region
2. Click **Copy to Rule** to capture the binary pixel values as a new pattern
3. Edit as needed (set some cells to "any" for more general matching)

### Anchor Positioning

Anchors snap to a 3x3 sub-grid within each cell:

```
corner ─── edge ─── corner
  │                    │
edge ── center ── edge
  │                    │
corner ─── edge ─── corner
```

This sub-pixel positioning allows match positions to be reported at pixel boundaries, centers, or corners — important for precise geometric feature detection.

---

## Rule Format

Rules are stored as JSON arrays. Each rule object:

```json
{
  "id": 1,
  "w": 3,
  "h": 3,
  "cells": [
    ["white", "black", "white"],
    ["black", "black", "black"],
    ["white", "black", "white"]
  ],
  "anchors": [
    {"x": 1.5, "y": 1.5}
  ],
  "negative": true,
  "rotational": true,
  "mirror": false,
  "exclude": false
}
```

| Field        | Type       | Description                                                                 |
|--------------|------------|-----------------------------------------------------------------------------|
| `id`         | `number`   | Unique identifier                                                           |
| `w`, `h`     | `number`   | Pattern dimensions                                                          |
| `cells`      | `string[][]` | 2D array: `"black"`, `"white"`, or `"any"` per cell                      |
| `anchors`    | `object[]` | Hit positions as `{x, y}` in pattern coordinates (float, sub-pixel)         |
| `negative`   | `boolean`  | Also match with colors inverted                                             |
| `rotational` | `boolean`  | Also match at 90/180/270 degree rotations                                   |
| `mirror`     | `boolean`  | Also match horizontal reflection                                            |
| `exclude`    | `boolean`  | If true, this rule removes matches instead of adding them                   |

### Coordinate System

- Cell `(0,0)` is the top-left of the pattern
- Anchor `{x: 1.5, y: 1.5}` places the hit at the center of cell `(1,1)` in a 3x3 grid
- Anchor `{x: 1.0, y: 1.0}` places the hit at the top-left corner of cell `(1,1)`
- Integer coordinates land on cell boundaries; `.5` offsets land on cell centers

---

## Algorithm

### Pattern Matching

For each rule, the algorithm:

1. **Generate transforms**: Expand the base pattern into all enabled symmetry variants (rotations, mirror, negative). Each variant includes transformed cell values and transformed anchor positions.

2. **Slide and test**: For each transform, slide the pattern across the image at every valid position. At each position, test every non-"any" cell against the corresponding image pixel. A match requires all specified cells to agree.

3. **Collect anchors**: For each matching position, map the pattern's anchor coordinates to image coordinates and record them as hit positions.

4. **Apply exclusions**: After all inclusion rules have been scanned, scan all exclusion rules. Any position matched by an exclusion rule is removed from the final hit set.

### Complexity

For an image of size `W x H` and a rule of size `w x h` with `T` symmetry transforms:

- **Per rule**: `O(W * H * w * h * T)` — brute-force sliding window
- **With exclusion**: Two passes over the image (inclusion + exclusion)

This is intentionally simple — no spatial indexing or FFT optimization — keeping the implementation transparent and the code auditable.

### Transform Generation

The D4 dihedral group transforms are computed explicitly:

| Transform     | Cell mapping                          | Anchor mapping                    |
|---------------|---------------------------------------|-----------------------------------|
| Identity      | `cells[y][x]`                         | `(x, y)`                         |
| 90 CW        | `cells[h-1-x][y]` → new `w=h, h=w`  | `(h-y, x)`                       |
| 180          | `cells[h-1-y][w-1-x]`                | `(w-x, h-y)`                     |
| 270 CW       | `cells[x][w-1-y]` → new `w=h, h=w`  | `(y, w-x)`                       |
| Mirror        | `cells[y][w-1-x]`                     | `(w-x, y)`                       |

Mirror is applied to each rotation variant, yielding up to 8 geometric transforms. With negative (color inversion), this doubles to 16 total pattern variants per rule.

---

## Project Structure

```
pixel-topology-rules/
├── index.html              # Interactive demo (single-file, no dependencies)
├── docs/
│   └── rule-format.md      # JSON schema and coordinate system reference
├── examples/
│   ├── corners.json        # Corner detection rules (L-junctions)
│   ├── endpoints.json      # Stroke endpoint detection
│   └── junctions.json      # T-junction and crossing detection
├── LICENSE
└── README.md
```

---

## Technical Notes

- **Zero dependencies**: Single HTML file with inline CSS and JavaScript. No build step, no frameworks, no external assets.
- **Client-side only**: All processing happens in the browser. No data is sent anywhere.
- **Canvas rendering**: Uses HTML5 Canvas with `imageSmoothingEnabled = false` for pixel-perfect rendering at any zoom level.
- **Binarization**: Images are converted using weighted luminance: `L = 0.299R + 0.587G + 0.114B`, threshold at 128. This follows the ITU-R BT.601 standard.

---

## Browser Compatibility

Works in any modern browser with HTML5 Canvas and ES6 support:
- Chrome 60+
- Firefox 55+
- Safari 11+
- Edge 79+

---

## License

MIT License. See [LICENSE](LICENSE) for details.
