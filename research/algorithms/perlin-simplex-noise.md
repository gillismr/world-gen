# Perlin Noise & Simplex Noise

## Overview

Gradient noise algorithms that produce smooth, pseudo-random, continuous output across N-dimensional space. Perlin noise (1983) and its successor Simplex noise (2001) are the most widely used algorithms for procedural texture and terrain generation.

---

## Perlin Noise

### How It Works

1. Define a grid of integer lattice points.
2. Assign each lattice point a pseudo-random **gradient vector**.
3. For a query point P, find the surrounding lattice cell.
4. Compute the **dot product** of each corner's gradient with the vector from that corner to P.
5. **Interpolate** the dot products using a smooth fade curve.

### Fade Function

The fade function smooths interpolation to avoid visible grid artifacts:

```
fade(t) = 6t⁵ − 15t⁴ + 10t³
```

This is Ken Perlin's improved fade (2002); the original used `3t² − 2t³`.

### 2D Pseudocode

```python
def perlin(x, y):
    # Integer cell coordinates
    xi = floor(x) & 255
    yi = floor(y) & 255

    # Fractional position within cell
    xf = x - floor(x)
    yf = y - floor(y)

    # Fade curves
    u = fade(xf)
    v = fade(yf)

    # Hash corner coordinates
    aa = perm[perm[xi    ] + yi    ]
    ab = perm[perm[xi    ] + yi + 1]
    ba = perm[perm[xi + 1] + yi    ]
    bb = perm[perm[xi + 1] + yi + 1]

    # Interpolate dot products
    x1 = lerp(grad(aa, xf,   yf  ),
               grad(ba, xf-1, yf  ), u)
    x2 = lerp(grad(ab, xf,   yf-1),
               grad(bb, xf-1, yf-1), u)
    return lerp(x1, x2, v)

def fade(t):
    return t * t * t * (t * (t * 6 - 15) + 10)

def lerp(a, b, t):
    return a + t * (b - a)

def grad(hash, x, y):
    # Map hash to one of 4 gradient vectors
    h = hash & 3
    u = x if h < 2 else y
    v = y if h < 2 else x
    return (u if (h & 1) == 0 else -u) + (v if (h & 2) == 0 else -v)
```

### Fractal Brownian Motion (fBm)

Raw Perlin noise is one frequency ("one octave"). To produce natural-looking terrain, sum multiple octaves at increasing frequencies and decreasing amplitudes:

```python
def fbm(x, y, octaves=6, lacunarity=2.0, gain=0.5):
    value = 0.0
    amplitude = 1.0
    frequency = 1.0
    for _ in range(octaves):
        value += amplitude * perlin(x * frequency, y * frequency)
        amplitude *= gain        # halve amplitude each octave
        frequency *= lacunarity  # double frequency each octave
    return value
```

- **Lacunarity**: frequency multiplier per octave (typically 2.0)
- **Gain / persistence**: amplitude multiplier per octave (typically 0.5)
- **Octaves**: how many layers to sum (6–8 is common for terrain)

---

## Simplex Noise

Ken Perlin's 2001 improvement over classical Perlin noise.

### Key Differences from Perlin

| Property | Perlin | Simplex |
|---|---|---|
| Grid shape | Hypercube (squares in 2D) | Simplex (triangles in 2D, tetrahedra in 3D) |
| Corners per cell | 2ⁿ | n+1 |
| Complexity | O(2ⁿ) | O(n²) |
| Directional artifacts | More visible | Fewer |
| Patent status | Public domain | Patent expired 2022 |

### 2D Simplex — Coordinate Skewing

```python
def simplex2(x, y):
    # Skew input to simplex grid
    F2 = 0.5 * (sqrt(3) - 1)
    s = (x + y) * F2
    i, j = floor(x + s), floor(y + s)

    # Unskew back
    G2 = (3 - sqrt(3)) / 6
    t = (i + j) * G2
    x0 = x - (i - t)
    y0 = y - (j - t)

    # Determine simplex (upper or lower triangle)
    if x0 > y0:
        i1, j1 = 1, 0   # lower triangle
    else:
        i1, j1 = 0, 1   # upper triangle

    x1 = x0 - i1 + G2
    y1 = y0 - j1 + G2
    x2 = x0 - 1 + 2*G2
    y2 = y0 - 1 + 2*G2

    # Hash corners
    ii, jj = i & 255, j & 255
    gi0 = perm[ii       + perm[jj      ]] % 12
    gi1 = perm[ii + i1  + perm[jj + j1 ]] % 12
    gi2 = perm[ii + 1   + perm[jj + 1  ]] % 12

    # Contribution from each corner
    def corner_contrib(gx, gy, gi):
        t = 0.5 - gx*gx - gy*gy
        if t < 0:
            return 0.0
        t *= t
        return t * t * dot(grad3[gi], gx, gy)

    n = (corner_contrib(x0, y0, gi0) +
         corner_contrib(x1, y1, gi1) +
         corner_contrib(x2, y2, gi2))
    return 70 * n   # scale to [-1, 1]
```

---

## Domain Warping

A powerful technique: feed the output of one noise function as the input coordinates of another. Produces swirling, organic shapes ideal for coastlines and cloud formations.

```python
def warped(x, y):
    # Warp coordinates by noise offset
    wx = fbm(x + 0.0, y + 0.0)
    wy = fbm(x + 5.2, y + 1.3)
    return fbm(x + 4.0*wx, y + 4.0*wy)
```

Reference: Inigo Quilez, "Warping" — https://iquilezles.org/articles/warp/

---

## References

- Ken Perlin's original 1985 SIGGRAPH paper: https://dl.acm.org/doi/10.1145/325165.325247
- Ken Perlin's improved noise (2002): https://cs.nyu.edu/~perlin/noise/
- Wikipedia — Perlin noise: https://en.wikipedia.org/wiki/Perlin_noise
- Wikipedia — Simplex noise: https://en.wikipedia.org/wiki/Simplex_noise
- Inigo Quilez noise articles: https://iquilezles.org/articles/
- Red Blob Games — noise overview: https://www.redblobgames.com/maps/terrain-from-noise/

---

## Application in Our World-Gen Pipeline

| Use | Layer | Details |
|---|---|---|
| **Heightmap generation** | Geology | fBm noise is the baseline for continent/ocean elevation |
| **Sea floor topography** | Geology | Separate noise pass with different parameters |
| **Temperature variation** | Climate | Noise-perturbed latitude-based gradient |
| **Precipitation variation** | Climate | Noise overlay on wind/rain shadow model |
| **Cloud cover / weather** | Climate | Animated 3D simplex noise |
| **Cave systems** | Geology | 3D Perlin threshold map (voxel worlds) |
| **Biome blending** | Biomes | Noise-offset boundaries to avoid hard edges |
| **Coastline irregularity** | Geology | Domain-warped noise for organic shapes |
| **Star/asteroid field placement** | Astronomy | 3D noise for nebula density, cluster distribution |

Noise is almost always the **starting material** that other simulation passes (erosion, tectonics, climate) refine into physically plausible output.
