# N-Body Simulation & Orbital Mechanics

## Overview

N-body simulation models the gravitational interactions between N massive bodies (stars, planets, moons, asteroids). It determines stable orbital configurations for solar systems — what planets exist, where they are, whether orbits are stable, and what kind of environments they produce (habitable zones, tidal locking, etc.).

For procedural world generation, orbital mechanics determines everything about the astronomical layer: stellar type, number and position of planets, day length, year length, axial tilt, and whether the planet is habitable.

---

## Gravitational Physics

Newton's law of universal gravitation:

```
F = G * m₁ * m₂ / r²
```

Where:
- `G = 6.674 × 10⁻¹¹ m³ kg⁻¹ s⁻²` (gravitational constant)
- `m₁, m₂` = masses of the two bodies
- `r` = distance between centers

The force vector from body j on body i:

```
F⃗ᵢⱼ = G * mᵢ * mⱼ * (r⃗ⱼ - r⃗ᵢ) / |r⃗ⱼ - r⃗ᵢ|³
```

Each body's acceleration:

```
a⃗ᵢ = Σⱼ≠ᵢ  G * mⱼ * (r⃗ⱼ - r⃗ᵢ) / |r⃗ⱼ - r⃗ᵢ|³
```

---

## Numerical Integration

Because the N-body problem has no closed-form solution for N ≥ 3, we integrate numerically.

### Euler Integration (Simple, Inaccurate)

```python
def euler_step(bodies, dt):
    for i, bi in enumerate(bodies):
        ax = ay = az = 0.0
        for j, bj in enumerate(bodies):
            if i == j:
                continue
            dx = bj.x - bi.x
            dy = bj.y - bi.y
            dz = bj.z - bi.z
            r3 = (dx**2 + dy**2 + dz**2)**1.5 + 1e-10  # softening
            f = G * bj.mass / r3
            ax += f * dx
            ay += f * dy
            az += f * dz
        bi.vx += ax * dt
        bi.vy += ay * dt
        bi.vz += az * dt

    for bi in bodies:
        bi.x += bi.vx * dt
        bi.y += bi.vy * dt
        bi.z += bi.vz * dt
```

### Leapfrog / Verlet Integration (Recommended — Symplectic)

Leapfrog is preferred for N-body because it **conserves energy** over long timescales (symplectic integrator):

```python
def leapfrog_step(bodies, dt):
    # Half-step velocity update
    accelerations = compute_accelerations(bodies)
    for bi, (ax, ay, az) in zip(bodies, accelerations):
        bi.vx += 0.5 * ax * dt
        bi.vy += 0.5 * ay * dt
        bi.vz += 0.5 * az * dt

    # Full-step position update
    for bi in bodies:
        bi.x += bi.vx * dt
        bi.y += bi.vy * dt
        bi.z += bi.vz * dt

    # Recompute accelerations at new positions
    accelerations = compute_accelerations(bodies)

    # Second half-step velocity update
    for bi, (ax, ay, az) in zip(bodies, accelerations):
        bi.vx += 0.5 * ax * dt
        bi.vy += 0.5 * ay * dt
        bi.vz += 0.5 * az * dt

def compute_accelerations(bodies):
    accs = [(0.0, 0.0, 0.0)] * len(bodies)
    for i, bi in enumerate(bodies):
        ax = ay = az = 0.0
        for j, bj in enumerate(bodies):
            if i == j:
                continue
            dx = bj.x - bi.x
            dy = bj.y - bi.y
            dz = bj.z - bi.z
            r2 = dx**2 + dy**2 + dz**2 + 1e-10   # softening to avoid singularity
            r3 = r2**1.5
            f = G * bj.mass / r3
            ax += f * dx
            ay += f * dy
            az += f * dz
        accs[i] = (ax, ay, az)
    return accs
```

Time complexity: O(N²) per step. For large N, Barnes-Hut tree reduces to O(N log N).

---

## Orbital Parameters (Kepler Elements)

For a two-body system (planet + star), stable orbits are described by:

| Parameter | Symbol | Description |
|---|---|---|
| Semi-major axis | a | Average orbital radius |
| Eccentricity | e | Shape: 0=circle, <1=ellipse, 1=parabola |
| Inclination | i | Tilt relative to reference plane |
| Longitude of ascending node | Ω | Rotation of orbit in plane |
| Argument of periapsis | ω | Where in orbit perihelion occurs |
| True anomaly | ν | Current position in orbit |

Orbital period (Kepler's 3rd Law):

```
T² = (4π² / G(M + m)) * a³

For star mass M >> planet mass m:
T = 2π * sqrt(a³ / (G * M))
```

---

## Habitable Zone (Goldilocks Zone)

The range of orbital radii where liquid water can exist on the surface:

```python
def habitable_zone(luminosity_solar):
    """
    luminosity_solar: star luminosity as multiple of Sun's (L☉)
    Returns: (inner_AU, outer_AU) habitable zone boundaries
    """
    # Kopparapu et al. (2013) — "runaway greenhouse" and "maximum greenhouse" limits
    inner_AU = math.sqrt(luminosity_solar / 1.1)   # runaway greenhouse
    outer_AU  = math.sqrt(luminosity_solar / 0.53)  # maximum greenhouse
    return inner_AU, outer_AU

def equilibrium_temperature(luminosity_solar, orbital_radius_AU, albedo=0.3):
    """Surface temperature estimate in Kelvin (no atmosphere)."""
    L = luminosity_solar * 3.828e26   # watts
    a = orbital_radius_AU * 1.496e11  # meters
    sigma = 5.67e-8  # Stefan-Boltzmann
    T = (L * (1 - albedo) / (16 * math.pi * sigma * a**2)) ** 0.25
    return T
```

---

## Accrete / Stellar System Formation

The Accrete algorithm (Dole 1969; Burdick 1988) simulates planetary system formation via dust/gas accretion around a star:

```
1. Generate a protoplanetary disk of dust and gas
2. Place planetesimal seeds at random orbital radii (log-uniform distribution)
3. Each seed sweeps an annular zone based on its mass:
   inner_sweep = a * (1 - e) - effect_radius
   outer_sweep = a * (1 + e) + effect_radius
4. Accrete dust/gas within sweep zone into the planetesimal
5. Repeat until no dust remains
6. Check if any planetesimal accreted enough gas to become a gas giant
7. Determine final planet compositions (rocky, icy, gas, water worlds)
```

```python
def accrete_system(stellar_mass):
    """Simplified Dole/Burdick accretion."""
    # Dust density: σ(a) = σ₀ * a^(-3/2) * (1 if a > snow_line else 1/3 for gas)
    snow_line = 5.0 * math.sqrt(stellar_mass)  # AU

    # Generate planetesimal nuclei
    nuclei = sorted([10**random.uniform(-2, 2) for _ in range(20)])  # AU

    # Iterative sweeping (simplified)
    planets = []
    for a in nuclei:
        mass = accrete_at_radius(a, stellar_mass, snow_line)
        if mass > 0.001:  # Earth masses
            planet_type = classify_planet(mass, a, snow_line)
            planets.append({'a': a, 'mass': mass, 'type': planet_type})

    return check_orbital_stability(planets)

def classify_planet(mass_earth, radius_AU, snow_line):
    if mass_earth > 50:
        return "Gas Giant"
    elif mass_earth > 10:
        return "Ice Giant" if radius_AU > snow_line else "Neptune-like"
    elif radius_AU > snow_line:
        return "Icy / Water World"
    else:
        return "Rocky / Terrestrial"
```

---

## Tidal Locking

A planet close to its star may become **tidally locked** (one face always facing the star, like Mercury or the Moon):

```python
def tidal_locking_timescale_years(planet_radius_m, planet_mass_kg,
                                   orbital_radius_m, star_mass_kg,
                                   rigidity=3e10):  # Pa (rock)
    G = 6.674e-11
    # Goldreich & Soter (1966) estimate
    t = (rigidity * orbital_radius_m**6 * planet_mass_kg) / \
        (3 * G * star_mass_kg**2 * planet_radius_m**3)
    t_years = t / (3.156e7)
    return t_years
```

A tidally locked planet has extreme climate: one side is permanent day, the other permanent night, with extreme temperature gradients. This dramatically affects the climate model.

---

## References

- Wikipedia — N-body simulation: https://en.wikipedia.org/wiki/N-body_simulation
- Wikipedia — Orbital mechanics: https://en.wikipedia.org/wiki/Orbital_mechanics
- Wikipedia — Kepler's laws: https://en.wikipedia.org/wiki/Kepler%27s_laws_of_planetary_motion
- Kopparapu et al. (2013) habitable zone: https://arxiv.org/abs/1301.6674
- Dole (1969) accretion: https://www.rand.org/content/dam/rand/pubs/research_memoranda/2005/RM5044.pdf
- Accrete source (modern Python): https://github.com/zakski/accrete-starform-stargen
- Universe Sandbox (interactive): https://universesandbox.com/

---

## Application in Our World-Gen Pipeline

The astronomical layer runs **first**, before any surface simulation.

| Output | Used By |
|---|---|
| **Stellar type + luminosity** | Habitable zone, UV radiation, star color in sky |
| **Orbital radius** | Year length, seasons, temperature baseline |
| **Axial tilt** | Season intensity (0° = no seasons, 90° = extreme) |
| **Orbital eccentricity** | Temperature variation over year |
| **Tidal locking** | Extreme climate model (permanent day/night sides) |
| **Moons** | Tidal forces, ocean tides, calendar for civilizations |
| **Multiple stars** | Complex day/night cycles, irregular seasons |
| **Year length** | Civilization calendar, agriculture cycles |
| **Day length** | Climate (slow rotation → larger temp swings) |

The system gives the climate simulation its **boundary conditions** — the energy input (stellar flux) that drives all atmospheric and surface processes.
