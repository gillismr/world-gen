# Whittaker Biome Classification

## Overview

The Whittaker biome classification system (Robert H. Whittaker, 1962/1975) maps **mean annual temperature** and **mean annual precipitation** to biome types. It is the most widely used algorithm for biome assignment in procedural world generation due to its simplicity and reasonable accuracy — two scalar inputs produce one categorical output (biome label).

---

## The Whittaker Diagram

The core of the system is a 2D lookup: given (temperature, precipitation), return a biome.

```
Mean Annual Precipitation (cm/yr)
   ^
450|                        Tropical
   |                     Rainforest
300|             Temperate
   |            Rainforest    Tropical
200|  Boreal                Seasonal
   | (Taiga)   Temperate      Forest
100|          Deciduous
   |            Forest    Savanna
 50| Tundra  Woodland/
   |         Shrubland
 25|         Temperate   Grassland
   |         Grassland
  0|  Ice   Desert                  Desert
   +---+-----+-----+------+------+-------> Mean Annual Temp (°C)
      -15   -5     5     15     20     25+
```

### Biome Lookup Table

| Temperature (°C) | Precipitation (cm/yr) | Biome |
|---|---|---|
| < -5 | any | **Polar / Ice** |
| -5 to 5 | any | **Tundra** |
| -5 to 5 | > 40 | **Boreal Forest (Taiga)** |
| 5 to 12 | < 30 | **Temperate Grassland** |
| 5 to 12 | 30–90 | **Temperate Deciduous Forest** |
| 5 to 12 | > 90 | **Temperate Rainforest** |
| 12 to 20 | < 50 | **Woodland / Shrubland** |
| 12 to 20 | 50–150 | **Tropical Seasonal Forest** |
| 20+ | < 25 | **Subtropical / Hot Desert** |
| 20+ | 25–75 | **Savanna** |
| 20+ | 75–150 | **Tropical Seasonal Forest** |
| 20+ | > 150 | **Tropical Rainforest** |

---

## Algorithm Pseudocode

```python
def whittaker_biome(temp_c, precip_cm):
    """
    temp_c:    mean annual temperature in Celsius
    precip_cm: mean annual precipitation in cm/year
    Returns:   biome name string
    """
    if temp_c < -5:
        return "Polar / Ice"

    if temp_c < 5:
        if precip_cm > 40:
            return "Boreal Forest (Taiga)"
        return "Tundra"

    if temp_c < 12:
        if precip_cm < 30:
            return "Temperate Grassland"
        elif precip_cm < 90:
            return "Temperate Deciduous Forest"
        else:
            return "Temperate Rainforest"

    if temp_c < 20:
        if precip_cm < 50:
            return "Woodland / Shrubland (Mediterranean)"
        elif precip_cm < 150:
            return "Tropical Seasonal Forest"
        else:
            return "Tropical Rainforest"

    # temp >= 20 (tropical/subtropical)
    if precip_cm < 25:
        return "Hot Desert"
    elif precip_cm < 75:
        return "Savanna"
    elif precip_cm < 150:
        return "Tropical Seasonal Forest"
    else:
        return "Tropical Rainforest"
```

---

## Extended Whittaker with Fuzzy Blending

Hard boundaries produce sharp, unnatural biome edges. Fuzzy/soft biome assignment uses **weighted distances from biome centroids** in (temp, precip) space to blend neighboring biomes at transitions:

```python
BIOME_CENTROIDS = {
    "Tropical Rainforest":      (26, 250),
    "Tropical Seasonal Forest": (22, 100),
    "Savanna":                  (22,  50),
    "Hot Desert":               (25,  15),
    "Subtropical Desert":       (20,  15),
    "Woodland / Shrubland":     (15,  40),
    "Temperate Rainforest":     (10, 140),
    "Temperate Deciduous":      ( 8,  65),
    "Temperate Grassland":      ( 5,  25),
    "Boreal Forest (Taiga)":    ( 0,  50),
    "Tundra":                   (-5,  20),
    "Polar / Ice":              (-15, 10),
}

def biome_weights(temp, precip, falloff=200.0):
    """Returns dict of {biome: weight} summing to 1.0."""
    weights = {}
    for biome, (bt, bp) in BIOME_CENTROIDS.items():
        # Normalize axes differently — precip range is larger
        dt = (temp - bt) / 30.0
        dp = (precip - bp) / 250.0
        dist_sq = dt**2 + dp**2
        weights[biome] = math.exp(-falloff * dist_sq)
    total = sum(weights.values())
    return {b: w/total for b, w in weights.items()}
```

The returned weights can then be used to blend biome colors, vegetation density, or other properties.

---

## Integration with Altitude

Temperature decreases with altitude (the **environmental lapse rate**):

```
temp_adjusted = temp_sea_level - (elevation_m / 1000.0) * 6.5  # 6.5°C per 1000m
```

This automatically creates alpine/montane zones at high elevations even in tropical regions, without any special-casing.

---

## Comparison: Whittaker vs. Köppen

| Property | Whittaker | Köppen |
|---|---|---|
| Inputs | Temp + Precip (annual means only) | Temp + Precip (monthly distributions) |
| Output categories | ~12 biomes | 30+ climate sub-zones |
| Complexity | Simple 2-input lookup | Requires full 12-month data |
| Typical use | Game/proc-gen (good enough) | Scientific modeling, detailed sim |
| Seasonal awareness | None | Full (distinguishes monsoon vs. Mediterranean) |

---

## References

- Whittaker, R.H. (1975) *Communities and Ecosystems*, 2nd ed. — original source
- Wikipedia — Biome: https://en.wikipedia.org/wiki/Biome
- Wikipedia — Whittaker biome classification: https://en.wikipedia.org/wiki/Biome#Whittaker_(1962,_1970,_1975)_biome-types
- WorldEngine biome code (uses Whittaker): https://github.com/Mindwerks/worldengine/blob/master/worldengine/biome.py

---

## Application in Our World-Gen Pipeline

Whittaker runs **after climate simulation** produces per-cell temperature and precipitation values.

| Output | Used By |
|---|---|
| **Biome map** | Rendering (color/texture), ecology layer |
| **Vegetation type** | Ecology, resource layer, civilization siting |
| **Treeline / snowline** | Rendering, game mechanics |
| **Soil fertility** | Agriculture potential for civilization layer |
| **Lumber / hunting resources** | Resource map for civilization economy |
| **Arability** | Farmland potential → settlement patterns |

For our project, the recommended approach is:
1. Use Köppen classification for full scientific accuracy (see `koppen-climate.md`)
2. Fall back to Whittaker for faster iteration or as a prototype step
3. Apply altitude correction via lapse rate
4. Use fuzzy blending to avoid hard biome edges in the renderer
