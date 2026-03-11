# L-Systems (Lindenmayer Systems)

## Overview

An L-system (Lindenmayer system) is a **parallel string-rewriting grammar** originally developed by Aristid Lindenmayer (1968) to model plant growth. A simple set of production rules, applied recursively, generates complex self-similar structures. In procedural generation, L-systems are used for vegetation, branching river networks, coastlines, cave systems, and city road layouts.

---

## Core Concepts

### Grammar

An L-system consists of:
- **Alphabet**: a set of symbols (e.g. `F`, `+`, `-`, `[`, `]`)
- **Axiom**: the starting string (e.g. `"F"`)
- **Production rules**: replacements applied simultaneously to every symbol (e.g. `F → FF+[+F-F-F]-[-F+F+F]`)

### Turtle Graphics Interpretation

The generated string is interpreted as instructions for a "turtle" that draws on a 2D (or 3D) canvas:

| Symbol | Meaning |
|---|---|
| `F` | Move forward (draw a line segment) |
| `f` | Move forward (no draw) |
| `+` | Turn left by angle δ |
| `-` | Turn right by angle δ |
| `[` | Push state (position + direction) onto stack |
| `]` | Pop state from stack |
| `\|` | Turn 180° |
| `&` | Pitch down (3D) |
| `^` | Pitch up (3D) |
| `<` | Roll left (3D) |
| `>` | Roll right (3D) |

---

## Algorithm

```python
def expand(axiom, rules, iterations):
    """Apply production rules for N iterations."""
    result = axiom
    for _ in range(iterations):
        next_result = []
        for char in result:
            next_result.append(rules.get(char, char))  # replace or keep unchanged
        result = ''.join(next_result)
    return result

def draw(instructions, step_length=10, angle_deg=25):
    """Interpret L-system string as turtle graphics. Returns list of line segments."""
    import math
    angle = math.radians(angle_deg)

    x, y = 0.0, 0.0
    heading = math.pi / 2  # pointing up
    stack = []
    segments = []

    for char in instructions:
        if char == 'F':
            nx = x + step_length * math.cos(heading)
            ny = y + step_length * math.sin(heading)
            segments.append(((x, y), (nx, ny)))
            x, y = nx, ny
        elif char == 'f':
            x += step_length * math.cos(heading)
            y += step_length * math.sin(heading)
        elif char == '+':
            heading += angle
        elif char == '-':
            heading -= angle
        elif char == '[':
            stack.append((x, y, heading))
        elif char == ']':
            x, y, heading = stack.pop()

    return segments
```

---

## Classic Examples

### Koch Snowflake
```
Axiom:   F--F--F
Rules:   F → F+F--F+F
Angle:   60°
```

### Sierpinski Triangle
```
Axiom:   F-G-G
Rules:   F → F-G+F+G-F
         G → GG
Angle:   120°
```

### Plant / Tree (Prusinkiewicz & Lindenmayer)
```
Axiom:   X
Rules:   X → F+[[X]-X]-F[-FX]+X
         F → FF
Angle:   25°
```

After 5 iterations, produces a realistic branching plant structure.

### More Organic Tree
```
Axiom:   F
Rules:   F → F[+F]F[-F]F
Angle:   25.7°
```

---

## Stochastic L-Systems

Instead of deterministic rules, use **probabilistic rules**. This breaks self-similar regularity and produces more organic shapes:

```python
STOCHASTIC_RULES = {
    'F': [
        (0.33, 'F[+F]F[-F]F'),
        (0.33, 'F[+F]F'),
        (0.34, 'F[-F]F')
    ]
}

def expand_stochastic(axiom, rules, iterations):
    import random
    result = axiom
    for _ in range(iterations):
        next_result = []
        for char in result:
            if char in rules:
                choices = rules[char]
                roll = random.random()
                cumulative = 0.0
                for prob, replacement in choices:
                    cumulative += prob
                    if roll < cumulative:
                        next_result.append(replacement)
                        break
            else:
                next_result.append(char)
        result = ''.join(next_result)
    return result
```

### Context-Sensitive L-Systems

Rules can depend on neighboring symbols in the string, enabling signals to propagate through a structure (like nutrients in a plant):

```
Rule: A < B > C → D   (replace B with D if it is preceded by A and followed by C)
```

This models apical dominance (how a tree's top branch suppresses lower branch growth) and is important for realistic tree architecture.

---

## Parametric L-Systems

Symbols carry parameters, enabling variable step lengths and angles:

```
Axiom:   A(1, 10)
Rules:   A(l, w) → F(l, w)[+A(l*0.7, w*0.7)][-A(l*0.7, w*0.7)]A(l*0.9, w*0.9)
         F(l, w) → forward(l) with thickness w
```

This produces branches that taper naturally as they grow thinner toward the tips.

---

## 3D L-Systems

Extend turtle graphics to 3D with pitch, yaw, and roll:

```python
def draw_3d(instructions, step=1.0, angle_deg=22.5):
    import numpy as np
    angle = math.radians(angle_deg)

    pos = np.array([0.0, 0.0, 0.0])
    # Heading vectors: H (forward), L (left), U (up)
    H = np.array([0.0, 1.0, 0.0])
    L = np.array([-1.0, 0.0, 0.0])
    U = np.array([0.0, 0.0, 1.0])

    stack = []
    segments = []

    def rotate(v, axis, a):
        """Rotate vector v around axis by angle a (Rodrigues' formula)."""
        return (v * math.cos(a)
                + np.cross(axis, v) * math.sin(a)
                + axis * np.dot(axis, v) * (1 - math.cos(a)))

    for char in instructions:
        if char == 'F':
            new_pos = pos + H * step
            segments.append((pos.copy(), new_pos.copy()))
            pos = new_pos
        elif char == '+': H, L = rotate(H, U, angle),  rotate(L, U, angle)
        elif char == '-': H, L = rotate(H, U, -angle), rotate(L, U, -angle)
        elif char == '&': H, U = rotate(H, L, angle),  rotate(U, L, angle)
        elif char == '^': H, U = rotate(H, L, -angle), rotate(U, L, -angle)
        elif char == '<': L, U = rotate(L, H, angle),  rotate(U, H, angle)
        elif char == '>': L, U = rotate(L, H, -angle), rotate(U, H, -angle)
        elif char == '[': stack.append((pos.copy(), H.copy(), L.copy(), U.copy()))
        elif char == ']': pos, H, L, U = stack.pop()

    return segments
```

---

## References

- Wikipedia — L-system: https://en.wikipedia.org/wiki/L-system
- Prusinkiewicz & Lindenmayer, *The Algorithmic Beauty of Plants* (1990) — **free PDF**: http://algorithmicbotany.org/papers/abop/abop.pdf
- Interactive L-system playground: https://morphocode.com/l-system-explorer/
- Paul Bourke's L-system examples: http://paulbourke.net/fractals/lsys/
- L-system trees in Houdini: https://www.sidefx.com/tutorials/l-systems/

---

## Application in Our World-Gen Pipeline

| Use | Layer | Details |
|---|---|---|
| **Tree / vegetation generation** | Ecology / Rendering | Per-species L-system rules generate 3D tree geometry |
| **Root systems** | Ecology | Underground L-systems for root spread (affects soil, water table) |
| **River branching networks** | Hydrology | Tributaries branch like a plant: main river is trunk, tributaries are branches |
| **Coastline fractal detail** | Geology/Rendering | Recursive refinement of polygon mesh coastlines |
| **Cave / tunnel systems** | Geology | 3D branching L-system for cave networks |
| **Road / street layout** | Cities | Parish (2001) used L-systems to generate city road networks organically |
| **Vein / reef patterns** | Ecology | Coral reef growth, mineral vein patterns in rock |

**Key insight:** River networks and L-systems are structurally identical — both are branching trees. The same L-system grammar that describes a plant also describes a river drainage basin. This means the river network can be both simulated physically (via erosion) and post-processed/rendered using L-system rules.

**Parish (2001) — Procedural Modeling of Cities** is the seminal paper using L-systems for city roads: https://graphics.cs.yale.edu/sites/default/files/p_roads.pdf
