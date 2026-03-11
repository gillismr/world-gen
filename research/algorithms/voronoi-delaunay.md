# Voronoi Diagrams & Delaunay Triangulation

## Overview

Voronoi diagrams and Delaunay triangulations are **dual data structures** — one is derived from the other — that partition space into regions based on proximity to a set of seed points. They are foundational to nearly every polygon-based map generation approach and appear throughout the world-gen literature.

---

## Voronoi Diagram

Given a set of seed points S in a plane, the **Voronoi diagram** partitions the plane into regions where every point in region Rᵢ is closer to seed sᵢ than to any other seed.

### Mathematical Definition

For seed points `s₁, s₂, ..., sₙ`:

```
V(sᵢ) = { p ∈ ℝ² : dist(p, sᵢ) ≤ dist(p, sⱼ) for all j ≠ i }
```

Where `dist` is typically Euclidean distance, though Manhattan or other metrics can be used for stylistic effects.

### Properties

- Each Voronoi **cell** contains exactly one seed point.
- Voronoi **edges** are equidistant from two seed points.
- Voronoi **vertices** are equidistant from three (or more) seed points.
- Cells are convex polygons.

### Pseudocode (Fortune's Algorithm — O(n log n))

Fortune's sweep-line algorithm is the efficient standard; a naive O(n³) approach simply tests each grid pixel:

```python
# Naive O(n * w * h) approach — correct but slow
def voronoi_naive(seeds, width, height):
    grid = [[0] * width for _ in range(height)]
    for y in range(height):
        for x in range(width):
            min_dist = float('inf')
            closest = 0
            for i, (sx, sy) in enumerate(seeds):
                d = (x - sx)**2 + (y - sy)**2  # skip sqrt for speed
                if d < min_dist:
                    min_dist = d
                    closest = i
            grid[y][x] = closest
    return grid
```

For production, use a library: `scipy.spatial.Voronoi` (Python), `d3-delaunay` (JS), or `spade` (Rust).

### Fortune's Algorithm (conceptual)

```
Sort events: site events (seed points) and circle events (Voronoi vertices)
Maintain a beach line (parabolic arcs of points equidistant from sweep line and each site)
Process events in order:
  - Site event: add new parabolic arc to beach line
  - Circle event: a parabolic arc shrinks to a point → emit a Voronoi vertex
Output: Voronoi cells and edges
```

Time complexity: O(n log n). Space: O(n).

---

## Delaunay Triangulation

The **dual** of the Voronoi diagram. Given the same seed points, connect seeds whose Voronoi cells share an edge. The result is a triangulation with the property that no seed point lies inside the circumcircle of any triangle.

### Delaunay Condition (Circumcircle Property)

For every triangle in the triangulation, its **circumscribed circle** (circumcircle) contains no other seed point:

```
For triangle (A, B, C):
  No point D exists such that D is inside circumcircle(A, B, C)
```

This condition maximizes the minimum angle of all triangles, avoiding thin "sliver" triangles.

### Incremental Insertion Pseudocode (Bowyer-Watson)

```python
def bowyer_watson(points):
    """Returns a list of (triangle) tuples as indices into points."""
    # Start with a super-triangle that contains all points
    super_tri = make_super_triangle(points)
    triangulation = [super_tri]

    for point in points:
        # Find all triangles whose circumcircle contains the new point
        bad_triangles = [t for t in triangulation if in_circumcircle(t, point)]

        # Find the boundary polygon of the bad triangles
        boundary = []
        for tri in bad_triangles:
            for edge in edges_of(tri):
                # Edge is on boundary if shared by exactly one bad triangle
                shared = sum(1 for t in bad_triangles if edge in edges_of(t))
                if shared == 1:
                    boundary.append(edge)

        # Remove bad triangles
        triangulation = [t for t in triangulation if t not in bad_triangles]

        # Re-triangulate with new point
        for edge in boundary:
            triangulation.append((edge[0], edge[1], point))

    # Remove triangles that share vertices with the super-triangle
    triangulation = [t for t in triangulation
                     if not any(v in super_tri for v in t)]
    return triangulation

def in_circumcircle(triangle, point):
    """Test if point lies inside the circumcircle of triangle."""
    ax, ay = triangle[0]
    bx, by = triangle[1]
    cx, cy = triangle[2]
    dx, dy = point
    # Determinant test
    matrix = [
        [ax-dx, ay-dy, (ax-dx)**2 + (ay-dy)**2],
        [bx-dx, by-dy, (bx-dx)**2 + (by-dy)**2],
        [cx-dx, cy-dy, (cx-dx)**2 + (cy-dy)**2],
    ]
    return determinant_3x3(matrix) > 0
```

---

## The Voronoi–Delaunay Duality

```
Voronoi → Delaunay:
  Connect seeds sharing a Voronoi edge → Delaunay triangle edge

Delaunay → Voronoi:
  Circumcenter of each Delaunay triangle → Voronoi vertex
  Connect adjacent circumcenters → Voronoi edge
```

In code, you typically compute one and derive the other. The `delaunator` library (Rust/JS) computes Delaunay triangulations in O(n log n) and is widely used.

---

## Relaxation (Lloyd's Algorithm)

Raw Poisson-disk or random seeds produce irregular, jagged Voronoi cells. **Lloyd's algorithm** relaxes the seeds to the **centroid** of their own cell, producing more uniform, natural-looking regions:

```python
def lloyds_relaxation(seeds, iterations=5):
    for _ in range(iterations):
        voronoi = compute_voronoi(seeds)
        seeds = [centroid(cell) for cell in voronoi.cells]
    return seeds
```

After a few iterations, the cells look much more like "natural" geographic regions.

---

## References

- Wikipedia — Voronoi diagram: https://en.wikipedia.org/wiki/Voronoi_diagram
- Wikipedia — Delaunay triangulation: https://en.wikipedia.org/wiki/Delaunay_triangulation
- Wikipedia — Fortune's algorithm: https://en.wikipedia.org/wiki/Fortune%27s_algorithm
- Amit Patel's polygon map generation: http://www-cs-students.stanford.edu/~amitp/game-programming/polygon-map-generation/
- mewo2 terrain algorithm: https://mewo2.com/notes/terrain/
- `delaunator` (Rust): https://crates.io/crates/delaunator
- `d3-delaunay` (JS): https://github.com/d3/d3-delaunay

---

## Application in Our World-Gen Pipeline

| Use | Layer | Details |
|---|---|---|
| **Continent / region cells** | Geology | Each Voronoi cell is a geographic region; seed with Poisson-disk points |
| **Plate tectonic boundaries** | Geology | Voronoi cells represent tectonic plates; edges are plate boundaries |
| **River networks** | Hydrology | Flow water "downhill" along Delaunay edges; rivers follow cell boundaries |
| **Political borders** | Civilization | Nation/territory borders follow Voronoi edges between city seeds |
| **Trade route graph** | Civilization | Delaunay triangulation gives the natural "neighbor graph" for road/trade networks |
| **Biome region assignment** | Biomes | Assign biome to each Voronoi cell based on elevation/temperature/moisture |
| **City neighborhood layout** | Cities | Subdivide city Voronoi cell into district polygons |
| **Climate sample regions** | Climate | Each Voronoi cell is a climate simulation tile |

Azgaar's Fantasy Map Generator and mewo2's terrain generator both use Voronoi/Delaunay as their primary data structure — essentially treating the map as a **polygon mesh** rather than a raster grid. This has big advantages: resolution-independent, irregular shapes look more natural, and neighbor relationships are explicit in the graph. This is likely the right approach for the core map representation in our pipeline.
