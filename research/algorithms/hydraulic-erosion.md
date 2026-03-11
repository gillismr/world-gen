# Hydraulic Erosion

## Overview

Hydraulic erosion simulates the physical process of water wearing away terrain over time. Rainwater flows downhill, picks up sediment, carries it, and deposits it elsewhere — carving valleys, smoothing mountains, and forming river deltas. It is the most important post-processing step after initial heightmap or tectonic simulation, dramatically increasing terrain realism.

There are two main approaches: **particle-based** (physically accurate, GPU-friendly) and **grid-based** (simpler, older).

---

## The Physical Process

1. **Rain** deposits water across the terrain.
2. Water flows downhill, forming **streams and rivers**.
3. Moving water has **erosive capacity** proportional to its speed and volume.
4. Water **picks up sediment** from the terrain.
5. When water slows (shallow slopes, pooling), it **deposits sediment**.
6. Over millions of iterations, valleys deepen, ridges sharpen, and sediment accumulates at the base of slopes (alluvial fans, river deltas).

---

## Particle-Based Hydraulic Erosion (Recommended)

Each raindrop is a particle that moves across the heightmap, carrying sediment. This approach is GPU-parallelizable and produces the most realistic results. Described by Hans Theobald Beyer (2015).

### Algorithm

```python
import math
import random

def hydraulic_erosion(heightmap, iterations=50000, params=None):
    if params is None:
        params = {
            'inertia':       0.05,   # [0,1] how much direction persists
            'capacity':      4.0,    # max sediment a droplet can carry
            'deposition':    0.3,    # how fast sediment is deposited
            'erosion':       0.3,    # how fast terrain is eroded
            'evaporation':   0.01,   # water loss per step
            'min_slope':     0.01,   # minimum slope for erosion
            'gravity':       4.0,    # acceleration down slope
            'lifetime':      30,     # max steps per droplet
            'radius':        3,      # erosion brush radius
        }

    w, h = len(heightmap[0]), len(heightmap)

    def height_and_gradient(x, y):
        # Bilinear interpolation of height and gradient at float position
        xi, yi = int(x), int(y)
        u, v = x - xi, y - yi
        # Clamp to grid
        xi = max(0, min(xi, w-2))
        yi = max(0, min(yi, h-2))

        h00 = heightmap[yi][xi]
        h10 = heightmap[yi][xi+1]
        h01 = heightmap[yi+1][xi]
        h11 = heightmap[yi+1][xi+1]

        height = h00*(1-u)*(1-v) + h10*u*(1-v) + h01*(1-u)*v + h11*u*v
        grad_x = (h10 - h00)*(1-v) + (h11 - h01)*v
        grad_y = (h01 - h00)*(1-u) + (h11 - h10)*u
        return height, grad_x, grad_y

    for _ in range(iterations):
        # Spawn droplet at random position
        x = random.uniform(1, w-2)
        y = random.uniform(1, h-2)
        vx, vy = 0.0, 0.0   # velocity
        water = 1.0
        sediment = 0.0

        for step in range(params['lifetime']):
            height, gx, gy = height_and_gradient(x, y)

            # Update direction (blend gradient with inertia)
            new_vx = vx * params['inertia'] - gx * (1 - params['inertia'])
            new_vy = vy * params['inertia'] - gy * (1 - params['inertia'])

            # Normalize
            speed = math.sqrt(new_vx**2 + new_vy**2)
            if speed < 1e-6:
                break   # droplet stalled
            new_vx /= speed
            new_vy /= speed

            vx, vy = new_vx, new_vy

            # Move droplet
            new_x = x + vx
            new_y = y + vy

            if not (0 < new_x < w-1 and 0 < new_y < h-1):
                break   # off the edge

            new_height, _, _ = height_and_gradient(new_x, new_y)
            delta_height = new_height - height

            # Sediment capacity depends on speed and slope
            slope = max(params['min_slope'], -delta_height)
            capacity = max(slope * speed * water * params['capacity'], 0.001)

            if sediment > capacity or delta_height > 0:
                # Deposit sediment (moving uphill or over capacity)
                deposit = (delta_height > 0
                           ? min(delta_height, sediment)   # fill pit
                           : (sediment - capacity) * params['deposition'])
                sediment -= deposit
                # Apply deposit to heightmap at current position (with brush)
                apply_to_heightmap(heightmap, x, y, deposit, params['radius'])
            else:
                # Erode terrain
                erode_amount = min((capacity - sediment) * params['erosion'],
                                   -delta_height)
                erode_amount = max(erode_amount, 0)
                sediment += erode_amount
                apply_to_heightmap(heightmap, x, y, -erode_amount, params['radius'])

            # Update velocity with gravity and evaporate water
            speed = math.sqrt(vx**2 + vy**2)
            vx = vx * speed + params['gravity'] * delta_height * vx / (speed + 1e-6)
            vy = vy * speed + params['gravity'] * delta_height * vy / (speed + 1e-6)
            water *= (1 - params['evaporation'])
            x, y = new_x, new_y

    return heightmap

def apply_to_heightmap(heightmap, x, y, delta, radius):
    """Apply erosion/deposition with a soft brush around (x, y)."""
    xi, yi = int(x), int(y)
    total_weight = 0.0
    weights = {}
    for dy in range(-radius, radius+1):
        for dx in range(-radius, radius+1):
            px, py = xi+dx, yi+dy
            if 0 <= px < len(heightmap[0]) and 0 <= py < len(heightmap):
                w = max(0, radius + 1 - math.sqrt(dx**2 + dy**2))
                weights[(px, py)] = w
                total_weight += w
    for (px, py), w in weights.items():
        heightmap[py][px] += delta * (w / total_weight)
```

---

## Grid-Based (Pipe Model) — Simpler Alternative

An older approach that operates on a full grid of water and sediment layers rather than individual particles.

```python
def grid_erosion_step(terrain, water, sediment, dt=0.01):
    """Single step of grid-based erosion."""
    h, w = len(terrain), len(terrain[0])
    new_water = [[water[y][x] for x in range(w)] for y in range(h)]
    new_sediment = [[sediment[y][x] for x in range(w)] for y in range(h)]

    for y in range(1, h-1):
        for x in range(1, w-1):
            total_h = terrain[y][x] + water[y][x]  # surface height
            outflow = 0.0
            for ny, nx in [(y-1,x),(y+1,x),(y,x-1),(y,x+1)]:
                neighbor_h = terrain[ny][nx] + water[ny][nx]
                delta = total_h - neighbor_h
                if delta > 0:
                    flow = min(delta * dt, water[y][x] / 4)
                    new_water[ny][nx] += flow
                    new_water[y][x]  -= flow
                    # Carry sediment proportionally
                    sed_flow = flow * sediment[y][x] / (water[y][x] + 1e-6)
                    new_sediment[ny][nx] += sed_flow
                    new_sediment[y][x]   -= sed_flow

    # Erosion and deposition
    for y in range(h):
        for x in range(w):
            erosion   = EROSION_RATE   * water[y][x]
            deposition = DEPOSIT_RATE  * sediment[y][x]
            terrain[y][x]   -= erosion - deposition
            new_sediment[y][x] += erosion - deposition

    return terrain, new_water, new_sediment
```

---

## Thermal Erosion

A simpler companion algorithm that simulates **gravity-driven slope collapse** (talus formation). Steep slopes shed material to lower neighbors:

```python
def thermal_erosion(heightmap, iterations=100, talus_angle=0.6):
    for _ in range(iterations):
        for y in range(1, len(heightmap)-1):
            for x in range(1, len(heightmap[0])-1):
                for ny, nx in [(y-1,x),(y+1,x),(y,x-1),(y,x+1)]:
                    diff = heightmap[y][x] - heightmap[ny][nx]
                    if diff > talus_angle:
                        transfer = (diff - talus_angle) * 0.5
                        heightmap[y][x]   -= transfer
                        heightmap[ny][nx] += transfer
    return heightmap
```

---

## GPU Acceleration

Particle-based erosion is **embarrassingly parallel** — each droplet is independent. This makes it ideal for GPU compute shaders. On a GPU, 500,000+ iterations can run in seconds vs. minutes on CPU.

In Rust/WGPU or WebGPU:
- Dispatch one compute thread per droplet
- Atomic operations needed for heightmap writes (or use double-buffering)
- Use shared memory for the erosion brush neighborhood

---

## References

- Hans Theobald Beyer (2015) "Implementation of a method for hydraulic erosion": https://www.firespark.de/resources/downloads/implementation%20of%20a%20methode%20for%20hydraulic%20erosion.pdf
- Sebastian Lague (2019) YouTube series on hydraulic erosion (highly recommended): https://www.youtube.com/watch?v=eaXk97uvbpU
- Wikipedia — Erosion: https://en.wikipedia.org/wiki/Erosion
- Wikipedia — Fluvial geomorphology: https://en.wikipedia.org/wiki/Fluvial_geomorphology
- GPU erosion paper (Mei et al. 2007): https://hal.inria.fr/inria-00402079

---

## Application in Our World-Gen Pipeline

| Use | Layer | Details |
|---|---|---|
| **Valley carving** | Geology | Rivers cut valleys into tectonic heightmap |
| **Mountain softening** | Geology | Raw tectonic mountains are sharp; erosion rounds them |
| **River channel formation** | Hydrology | Erosion paths become persistent river channels |
| **Alluvial plains / deltas** | Geology | Sediment deposits at slope breaks → flat fertile land |
| **Coastal cliffs vs. beaches** | Geology | Wave erosion (simplified) shapes coastline type |
| **Soil depth map** | Biomes | Deposited sediment = deep fertile soil; eroded rock = thin soils |
| **Desert dune formation** | Biomes | Wind erosion (Aeolian) variant for arid regions |

Erosion runs **after tectonics** and **before climate simulation**. The output — a refined heightmap plus a sediment/soil depth map — directly feeds into:
- River pathfinding (follow erosion channels)
- Biome assignment (alluvial plains → grassland/farmland)
- Civilization placement (fertile river valleys are prime settlement sites)
