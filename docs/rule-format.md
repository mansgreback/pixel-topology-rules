# Rule Format Reference

## JSON Schema

Rules are stored as a JSON array. Each element describes one pattern rule:

```json
[
  {
    "id": 1,
    "w": 3,
    "h": 3,
    "cells": [
      ["white", "black", "any"],
      ["black", "black", "any"],
      ["any",   "any",   "any"]
    ],
    "anchors": [
      {"x": 1.5, "y": 1.5}
    ],
    "negative": true,
    "rotational": true,
    "mirror": false,
    "exclude": false
  }
]
```

## Fields

### `id` (number)

Unique integer identifier. Used internally for tracking which rules produced which hits. When loading from file, IDs are preserved. When creating new rules, the next available ID is assigned automatically.

### `w`, `h` (number)

Width and height of the pattern grid. Minimum 2, maximum 12 in each dimension. The pattern does not need to be square.

### `cells` (string[][])

2D array of size `h` rows by `w` columns. Each cell is one of:

| Value     | Meaning                                        |
|-----------|------------------------------------------------|
| `"black"` | Must match a black (foreground) pixel          |
| `"white"` | Must match a white (background) pixel          |
| `"any"`   | Matches either color (wildcard / don't care)   |

A pattern with all cells set to `"any"` matches everywhere and is treated as empty.

### `anchors` (object[])

Array of `{x, y}` objects specifying where to report a hit when the pattern matches. Coordinates are in pattern-space (not image-space) using a continuous coordinate system:

```
 0    0.5    1    1.5    2    2.5    3
 ├─────┼─────┼─────┼─────┼─────┼─────┤
 │  cell 0   │  cell 1   │  cell 2   │
```

- **Integer values** (0, 1, 2, ...) land on cell boundaries
- **Half-integer values** (0.5, 1.5, ...) land on cell centers
- A value of `0` is the left/top edge of the pattern
- A value equal to `w` or `h` is the right/bottom edge

When a pattern matches at image position `(ix, iy)`, each anchor at `(ax, ay)` reports a hit at `(ix + ax, iy + ay)` in image coordinates.

#### Sub-pixel anchor grid

Within each cell, anchors snap to 9 positions:

```
(x, y)──────(x+0.5, y)──────(x+1, y)
  │                              │
(x, y+0.5)  (x+0.5, y+0.5)  (x+1, y+0.5)
  │                              │
(x, y+1)────(x+0.5, y+1)────(x+1, y+1)
```

This allows reporting hits at pixel centers (`.5` offsets), pixel boundaries (integer offsets), or pixel corners.

### `negative` (boolean)

When `true`, the pattern also matches with all colors inverted:
- `"black"` cells match white pixels
- `"white"` cells match black pixels
- `"any"` cells still match anything

This effectively doubles the number of pattern variants tested.

### `rotational` (boolean)

When `true`, the pattern matches at 0, 90, 180, and 270 degree clockwise rotations. For non-square patterns (w != h), the 90 and 270 rotations swap width and height.

Anchor coordinates are transformed along with the cell grid.

### `mirror` (boolean)

When `true`, horizontal mirror reflections are also tested. Mirror is applied to each rotation variant, so with both `rotational` and `mirror` enabled, up to 8 geometric transforms are tested.

### `exclude` (boolean)

When `true`, this rule acts as an exclusion rule:
- Its matches are computed the same way as inclusion rules
- But instead of adding positions to the hit set, it **removes** any overlapping positions

Exclusion rules are applied after all inclusion rules, implementing a two-pass pipeline:

```
Pass 1: Union of all inclusion rule hits → candidate set
Pass 2: Union of all exclusion rule hits → removal set
Final: candidate set − removal set
```

## Transform Combinations

| negative | rotational | mirror | Max variants |
|----------|------------|--------|-------------|
| false    | false      | false  | 1           |
| true     | false      | false  | 2           |
| false    | true       | false  | 4           |
| false    | false      | true   | 2           |
| true     | true       | false  | 8           |
| true     | false      | true   | 4           |
| false    | true       | true   | 8           |
| true     | true       | true   | 16          |

Duplicate transforms (e.g., a symmetric pattern that looks the same after rotation) are automatically detected and removed based on the serialized cell+anchor representation.

## Example Rules

### Plus/cross detector (3x3)

```json
{
  "id": 1, "w": 3, "h": 3,
  "cells": [
    ["any",   "black", "any"],
    ["black", "black", "black"],
    ["any",   "black", "any"]
  ],
  "anchors": [{"x": 1.5, "y": 1.5}],
  "negative": true, "rotational": false, "mirror": false, "exclude": false
}
```

### L-corner detector (3x3)

```json
{
  "id": 2, "w": 3, "h": 3,
  "cells": [
    ["white", "white", "any"],
    ["white", "black", "black"],
    ["any",   "black", "any"]
  ],
  "anchors": [{"x": 1.5, "y": 1.5}],
  "negative": true, "rotational": true, "mirror": false, "exclude": false
}
```

### Horizontal stroke endpoint (3x3)

```json
{
  "id": 3, "w": 3, "h": 3,
  "cells": [
    ["white", "white", "white"],
    ["black", "black", "white"],
    ["white", "white", "white"]
  ],
  "anchors": [{"x": 1.5, "y": 1.5}],
  "negative": true, "rotational": true, "mirror": true, "exclude": false
}
```
