# Diamond-Square Algorithm (Midpoint Displacement)

## Overview

A classic fractal terrain generation algorithm that produces heightmaps with self-similar, mountain-like structure. It works on a 2D grid whose side length is `2ⁿ + 1` (e.g. 5×5, 9×9, 17×17, 65×65, 257×257...) and alternates between two passes — the **diamond step** and the **square step** — until every cell has a height value.

First described by Fournier, Fussell, and Carpenter (1982). Also called "plasma fractal" or "random midpoint displacement."

---

## Algorithm

### Initialization

Set the four corners of the grid to arbitrary (or random) seed values.

```
+---+---+---+---+---+
| a |   |   |   | b |
+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
| c |   |   |   | d |
+---+---+---+---+---+
```

### Diamond Step

For each square of side `s`, compute the **center** as the average of the four corners, plus a random offset scaled by the current roughness:

```
center = (a + b + c + d) / 4 + random(-r, r)
```

This gives a diamond pattern of newly filled points:

```
+---+---+---+---+---+
| a |   |   |   | b |
+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
|   |   | X |   |   |    X = diamond center
+---+---+---+---+---+
|   |   |   |   |   |
+---+---+---+---+---+
| c |   |   |   | d |
+---+---+---+---+---+
```

### Square Step

For each diamond (rotated square), compute the **midpoint of each edge** as the average of the two adjacent corners plus a random offset:

```
edge_midpoint = (adjacent_corners...) / count + random(-r, r)
```

Note: edge midpoints on the grid boundary have only 2 or 3 neighbors. Some implementations wrap the grid (tiling); others clamp or omit the missing neighbor.

### Roughness Decay

After each full diamond+square pass, halve the roughness `r`:

```
r = r * 2^(-H)    where H ∈ [0, 1] controls roughness (H=1 → smooth, H=0 → rough)
```

Commonly simplified to `r *= 0.5` each iteration.

### Complete Pseudocode

```python
import random
import math

def diamond_square(n, roughness=1.0, H=1.0):
    """
    n: grid is (2^n + 1) x (2^n + 1)
    roughness: initial random range
    H: Hurst exponent controlling fractal dimension (0=rough, 1=smooth)
    """
    size = 2**n + 1
    grid = [[0.0] * size for _ in range(size)]

    # Seed corners
    grid[0][0]           = random.uniform(-1, 1)
    grid[0][size-1]      = random.uniform(-1, 1)
    grid[size-1][0]      = random.uniform(-1, 1)
    grid[size-1][size-1] = random.uniform(-1, 1)

    step = size - 1
    scale = roughness

    while step > 1:
        half = step // 2

        # Diamond step
        for y in range(0, size-1, step):
            for x in range(0, size-1, step):
                avg = (grid[y][x] + grid[y][x+step] +
                       grid[y+step][x] + grid[y+step][x+step]) / 4.0
                grid[y+half][x+half] = avg + random.uniform(-scale, scale)

        # Square step
        for y in range(0, size, half):
            for x in range((y + half) % step, size, step):
                neighbors = []
                if y - half >= 0:    neighbors.append(grid[y-half][x])
                if y + half < size:  neighbors.append(grid[y+half][x])
                if x - half >= 0:    neighbors.append(grid[y][x-half])
                if x + half < size:  neighbors.append(grid[y][x+half])
                grid[y][x] = sum(neighbors) / len(neighbors) + random.uniform(-scale, scale)

        step = half
        scale *= 2**(-H)

    return grid
```

---

## Output Characteristics

- Values form a **fractal heightmap** with self-similar detail at all scales.
- The Hurst exponent `H` controls the fractal dimension:
  - `H = 1.0` → very smooth, gentle rolling hills
  - `H = 0.5` → moderate roughness (typical terrain)
  - `H = 0.0` → maximum roughness (jagged, spiky)
- The grid **must** be `2ⁿ + 1` in size; arbitrary sizes require patching.
- Visible **grid-aligned artifacts** at coarse scales — a known weakness vs. Perlin noise.

---

## Visual

```
Roughness H=0.8 (smooth):        Roughness H=0.3 (rough):

  ~~~~                              /\/\/\
 ~~~~~~                           /\/   \/\
~~~~~~                           /        \/\/
```

---

## Comparison to Perlin/Simplex Noise

| Property | Diamond-Square | Perlin / Simplex |
|---|---|---|
| Grid requirement | Must be 2ⁿ+1 | Any size |
| Tileable | With modifications | Yes (natively) |
| Grid artifacts | Noticeable | Minimal |
| Speed | Fast, O(n²) | Fast, O(n²) |
| Multi-octave | Built-in via recursion | fBm layering |
| Control | Less flexible | Very flexible |
| Typical use | Simple terrain demos | Production terrain |

Diamond-square is simpler to implement from scratch and good for quick prototypes; Perlin/Simplex is preferred for production.

---

## References

- Wikipedia: https://en.wikipedia.org/wiki/Diamond-square_algorithm
- Original paper: Fournier, Fussell, Carpenter (1982) — "Computer Rendering of Stochastic Models"
- Paul Bourke's explanation: http://paulbourke.net/fractals/noise/
- Red Blob Games terrain noise: https://www.redblobgames.com/maps/terrain-from-noise/

---

## Application in Our World-Gen Pipeline

| Use | Layer | Details |
|---|---|---|
| **Prototype heightmaps** | Geology | Quick-and-dirty continents for early testing |
| **Island / plateau generation** | Geology | Seed corners intentionally (e.g., high center → island) |
| **Texture detail** | Rendering | Roughness detail on rock/cliff surfaces |
| **Initial elevation before tectonics** | Geology | Cheap starting point that tectonic sim then reshapes |
| **Lunar/asteroid surface** | Astronomy | No tectonics → raw fractal surface is plausible |

In our pipeline, diamond-square is most valuable as a **fast initial scaffold** for the geology layer before physically based erosion and tectonics are applied. For the final product, Simplex fBm is likely preferred due to fewer artifacts and more control.
