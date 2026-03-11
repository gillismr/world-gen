# Plate Tectonics Simulation

## Overview

Plate tectonics simulation models the movement of large rigid sections of a planet's lithosphere ("plates") over geological time. The movement of plates creates mountains, ocean trenches, volcanoes, and determines the large-scale shape of continents. It is the most physically grounded approach to generating believable continental geometry and is the first major simulation step after initial crust formation.

---

## Real-World Context

- The Earth has ~15 major plates and many minor ones.
- Plates move at 1–15 cm/year (fast by geological standards).
- At plate boundaries, three things happen:
  - **Divergent** (spreading): plates move apart → rift valleys, mid-ocean ridges
  - **Convergent** (subduction/collision): plates push together → mountains, volcanoes, trenches
  - **Transform** (strike-slip): plates slide sideways → fault lines, earthquakes
- The simulation needs to run for hundreds of millions of years.

---

## Simulation Approaches

### 1. Discrete Plate / Cell-Based (Most Common in Games/Proc-Gen)

Each cell on a grid or Voronoi mesh is assigned to a plate. Plates are rigid bodies that move; where they collide, mountains form.

```python
class Plate:
    def __init__(self, id, seed_x, seed_y, velocity_x, velocity_y, is_oceanic):
        self.id = id
        self.seed = (seed_x, seed_y)
        self.vx = velocity_x    # angular velocity components
        self.vy = velocity_y
        self.is_oceanic = is_oceanic   # oceanic vs. continental crust
        self.cells = []         # list of grid cells belonging to this plate

class TectonicSimulation:
    def __init__(self, width, height, n_plates):
        self.width = width
        self.height = height
        self.elevation = [[0.0] * width for _ in range(height)]
        self.plate_map = [[None] * width for _ in range(height)]

        # Generate plates via Voronoi seeding
        seeds = poisson_disk_sample(n_plates, width, height)
        self.plates = [Plate(i, sx, sy,
                             random_velocity(), random_velocity(),
                             random() < 0.7)   # 70% oceanic
                       for i, (sx, sy) in enumerate(seeds)]
        self.assign_cells_to_plates()

    def assign_cells_to_plates(self):
        # Voronoi assignment: each cell goes to closest seed
        for y in range(self.height):
            for x in range(self.width):
                closest = min(self.plates,
                              key=lambda p: dist((x,y), p.seed))
                self.plate_map[y][x] = closest.id
                closest.cells.append((x, y))

    def step(self, dt):
        # Move each plate by its velocity (wrapping at edges)
        for plate in self.plates:
            plate.seed = ((plate.seed[0] + plate.vx * dt) % self.width,
                          (plate.seed[1] + plate.vy * dt) % self.height)

        # Re-detect boundaries and compute elevation changes
        self.process_boundaries()

    def process_boundaries(self):
        for y in range(self.height):
            for x in range(self.width):
                # Check 4-neighbors for plate boundary
                for nx, ny in neighbors(x, y):
                    if self.plate_map[y][x] != self.plate_map[ny][nx]:
                        self.process_boundary_pair(x, y, nx, ny)

    def process_boundary_pair(self, x1, y1, x2, y2):
        p1 = self.plates[self.plate_map[y1][x1]]
        p2 = self.plates[self.plate_map[y2][x2]]
        interaction = boundary_type(p1, p2)

        if interaction == 'convergent':
            if p1.is_oceanic and p2.is_oceanic:
                # Island arc + trench
                self.elevation[y1][x1] += ISLAND_ARC_UPLIFT
                self.elevation[y2][x2] -= TRENCH_DEPTH
            elif p1.is_oceanic != p2.is_oceanic:
                # Subduction: oceanic goes under continental → mountains + trench
                oceanic = p1 if p1.is_oceanic else p2
                continental = p2 if p1.is_oceanic else p1
                # Continental gains a mountain range
                continental_cell = (x1, y1) if not p1.is_oceanic else (x2, y2)
                self.elevation[continental_cell[1]][continental_cell[0]] += MOUNTAIN_UPLIFT
            else:
                # Continental collision → massive mountain range (Himalayas)
                self.elevation[y1][x1] += COLLISION_UPLIFT
                self.elevation[y2][x2] += COLLISION_UPLIFT

        elif interaction == 'divergent':
            # Rift valley or mid-ocean ridge
            self.elevation[y1][x1] -= RIFT_SUBSIDENCE
            self.elevation[y2][x2] -= RIFT_SUBSIDENCE
```

### 2. Bird (2003) Classification — Real Plate Boundary Types

The WorldEngine library follows classifications from Bird's 2003 paper on plate boundaries:

| Boundary Type | Abbrev | Formation |
|---|---|---|
| Continental rift | CRB | Two continental plates diverge |
| Mid-ocean ridge | OSR | Two oceanic plates diverge |
| Subduction (ocean-ocean) | OOC | Denser oceanic subducts |
| Subduction (ocean-continent) | OCB | Oceanic subducts under continental |
| Continental collision | CCB | Two continentals collide |
| Transform fault | CTF / OTF | Plates slide laterally |

### 3. Velocity / Stress Fields

More advanced simulations model plates as rigid bodies with angular velocity around a **Euler pole** (the pole of rotation for a plate on a sphere):

```
Velocity at point P on plate = ω × (P - Euler_pole)
```

Where `ω` is the angular velocity vector. This gives realistic plate motion on a spherical planet.

---

## Elevation Model

A common parameterization for elevation after N timesteps:

```
Continental crust:  base elevation = +200 to +500m (above sea)
Oceanic crust:      base elevation = -3000 to -5000m (below sea)

At convergent boundary (subduction):
  Mountain side:    +uplift_rate * years_of_convergence (capped ~8000m)
  Trench side:      -subsidence_rate * years

At divergent boundary:
  -rift_rate * years (typically forms a valley or ocean basin)

Isostasy (gravitational balance):
  elevation += -k * (elevation - target_isostatic_elevation)   # spring toward equilibrium
```

---

## Key Parameters

| Parameter | Typical Range | Effect |
|---|---|---|
| Number of plates | 6–20 | More plates → more complex coastlines |
| Plate speed | 1–15 cm/yr (scaled) | Faster → more dramatic features faster |
| Continental/oceanic ratio | 60–80% oceanic | More oceanic → more islands, archipelagos |
| Simulation timestep | 1–10 Myr | Finer steps → more accurate boundaries |
| Total time | 100–500 Myr | More time → more evolved terrain |

---

## References

- WorldEngine source (Python): https://github.com/Mindwerks/worldengine
- plate-tectonics C++ library: https://github.com/Mindwerks/plate-tectonics
- Bird (2003) "An updated digital model of plate boundaries": https://agupubs.onlinelibrary.wiley.com/doi/10.1029/2001GC000252
- Wikipedia — Plate tectonics: https://en.wikipedia.org/wiki/Plate_tectonics
- Wikipedia — Euler pole: https://en.wikipedia.org/wiki/Euler_pole
- Artifexian (YouTube) — Building a realistic world, plate tectonics episode

---

## Application in Our World-Gen Pipeline

This is the **most important simulation in the geology layer**. It determines:

| Output | Used By |
|---|---|
| Continent/ocean distribution | All subsequent layers |
| Mountain range locations | Climate (rain shadows), erosion, biomes |
| Volcanic regions | Geology detail, mineral resources, civilization siting |
| Ocean ridge/trench locations | Bathymetry, ocean current seed points |
| Fault line positions | Earthquake risk zones for civilization layer |
| Crustal age map | Ocean floor depth (older crust is denser → deeper) |

**Recommended approach for our project:** Start with Voronoi-seeded plates (10–15 plates), assign continental/oceanic type, simulate convergent/divergent boundaries with uplift/subsidence, then pass the elevation map to the erosion simulation.
