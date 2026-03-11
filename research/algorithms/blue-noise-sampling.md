# Blue Noise Sampling

## Overview

Blue noise is a type of **point distribution** that avoids clustering and regularity. Points are distributed such that no two points are too close together, and no large empty regions exist — the result looks like a natural, organic scatter rather than white noise (clumped) or a uniform grid (rigid).

The name comes from the power spectrum: blue noise has low energy at low frequencies (no large-scale clumping) and high energy at high frequencies (local variation), the opposite of red noise.

---

## Why It Matters

White noise (purely random) produces clusters and large voids that look artificial. Blue noise matches how many natural phenomena distribute themselves: trees in a forest, stones on a beach, cities across a landscape — all maintain rough minimum distances from each other while not forming perfect grids.

---

## Poisson Disk Sampling (Primary Algorithm)

The most common method for generating blue noise point sets is **Poisson disk sampling**, which enforces a minimum distance `r` between all pairs of points.

### Bridson's Algorithm (2007) — Fast Poisson Disk Sampling

Robert Bridson's algorithm generates Poisson disk samples in O(n) time.

```python
import random
import math

def poisson_disk_sampling(width, height, r, k=30):
    """
    width, height: dimensions of the sample region
    r: minimum distance between points
    k: max attempts before a point is considered 'done'
    Returns: list of (x, y) sample points
    """
    cell_size = r / math.sqrt(2)   # Each grid cell can hold at most one point
    cols = math.ceil(width  / cell_size)
    rows = math.ceil(height / cell_size)
    grid = [None] * (cols * rows)   # Spatial acceleration grid

    def grid_idx(x, y):
        return int(x / cell_size) + int(y / cell_size) * cols

    def in_bounds(x, y):
        return 0 <= x < width and 0 <= y < height

    def too_close(x, y):
        gx = int(x / cell_size)
        gy = int(y / cell_size)
        for dy in range(-2, 3):
            for dx in range(-2, 3):
                nx, ny = gx + dx, gy + dy
                if 0 <= nx < cols and 0 <= ny < rows:
                    p = grid[nx + ny * cols]
                    if p and math.dist((x, y), p) < r:
                        return True
        return False

    # Seed
    x0 = random.uniform(0, width)
    y0 = random.uniform(0, height)
    samples = [(x0, y0)]
    active = [(x0, y0)]
    grid[grid_idx(x0, y0)] = (x0, y0)

    while active:
        idx = random.randrange(len(active))
        px, py = active[idx]
        found = False

        for _ in range(k):
            # Generate candidate in annulus [r, 2r]
            angle = random.uniform(0, 2 * math.pi)
            dist  = random.uniform(r, 2 * r)
            cx = px + dist * math.cos(angle)
            cy = py + dist * math.sin(angle)

            if in_bounds(cx, cy) and not too_close(cx, cy):
                samples.append((cx, cy))
                active.append((cx, cy))
                grid[grid_idx(cx, cy)] = (cx, cy)
                found = True
                break

        if not found:
            active.pop(idx)

    return samples
```

**Time complexity:** O(n) where n is the number of output points.
**Space complexity:** O(n) for the grid.

---

## Mitchell's Best Candidate (Simpler Alternative)

A simpler but slower approach: for each new point, generate `k` random candidates and keep the one farthest from all existing points.

```python
def mitchells_best_candidate(width, height, n, k=10):
    points = []
    for _ in range(n):
        best = None
        best_dist = -1
        for _ in range(k):
            cx = random.uniform(0, width)
            cy = random.uniform(0, height)
            min_d = min((math.dist((cx, cy), p) for p in points), default=float('inf'))
            if min_d > best_dist:
                best_dist = min_d
                best = (cx, cy)
        points.append(best)
    return points
```

Slower than Bridson (O(n²k)) but easier to implement.

---

## Weighted / Variable-Density Blue Noise

A powerful extension: modulate the minimum distance `r` by a density map so that points cluster more densely in certain regions.

```python
def variable_density_poisson(width, height, density_map, r_min, r_max, k=30):
    """
    density_map: 2D array of floats in [0,1]; higher = denser placement
    r for a given position = lerp(r_max, r_min, density_map[x][y])
    """
    # Same as Bridson's, but r is sampled from density_map at each point's location
    ...
```

This is how you'd place **cities more densely near coasts** or **trees more sparsely at high altitude**.

---

## Power Spectrum Comparison

```
White Noise:    . .   .. .   .    (random, has clusters and voids)
Blue Noise:     .  .   .  .   .  (evenly spread, no clusters, no grid)
Grid:           .  .   .  .   .  (perfectly regular — too artificial)
              (equal spacing, rigid)
```

---

## References

- Bridson (2007) "Fast Poisson Disk Sampling in Arbitrary Dimensions": https://www.cs.ubc.ca/~rbridson/docs/bridson-siggraph07-poissondisk.pdf
- Wikipedia — Supersampling / Blue noise: https://en.wikipedia.org/wiki/Blue_noise
- Wikipedia — Poisson disk sampling: https://en.wikipedia.org/wiki/Supersampling#Poisson_disk
- Inigo Quilez on sampling: https://iquilezles.org/articles/
- Interactive demo: https://observablehq.com/@techsparx/fast-poisson-disk-sampling

---

## Application in Our World-Gen Pipeline

| Use | Layer | Details |
|---|---|---|
| **City/settlement placement** | Civilization | Minimum distance ensures cities aren't overlapping; density map weights toward rivers, coasts |
| **Forest / vegetation distribution** | Biomes | Trees spread via Poisson disk within a biome; density weighted by soil quality |
| **Mountain peak placement** | Geology | Seed Voronoi cells for tectonic plates or mountain ranges |
| **River source placement** | Geology/Hydrology | Distribute river headwaters naturally across highlands |
| **Volcano / hotspot placement** | Geology | Space volcanoes realistically across a plate |
| **Dungeon/POI scatter** | Game layer | Distribute points of interest on the game map |
| **Star / solar system placement** | Astronomy | Distribute star systems in a galaxy without crowding |
| **Sample points for climate sim** | Climate | Choose representative sampling locations for atmospheric simulation |

Blue noise sampling is used whenever you need to **scatter discrete objects across space** in a way that looks natural. It's the difference between a forest that looks real and one that looks hand-placed or randomly clumped. The combination with a density/weight map (weighted Poisson disk) makes it extremely powerful for our pipeline.
