# Köppen Climate Classification

## Overview

The Köppen climate classification system (Wladimir Köppen, 1884; updated by Rudolf Geiger, 1961) is the world's most widely used climate classification scheme. Unlike Whittaker, it uses **monthly temperature and precipitation data** — not just annual means — enabling it to capture seasonal patterns like monsoons, Mediterranean summers, and continental winters. This makes it significantly more accurate and complex than Whittaker.

---

## Classification Structure

The system uses a **letter code** built up hierarchically:

```
First letter (A/B/C/D/E) — Major group
Second letter             — Precipitation pattern
Third letter (optional)   — Temperature sub-type
```

### First Letter — Major Climate Groups

| Code | Name | Condition |
|---|---|---|
| **A** | Tropical | All months T > 18°C |
| **B** | Arid / Desert | Evaporation > Precipitation |
| **C** | Temperate | Coldest month -3°C to 18°C |
| **D** | Continental | Coldest month < -3°C, warmest > 10°C |
| **E** | Polar | Warmest month < 10°C |

### B — Arid Group Threshold

The aridity threshold uses annual precipitation vs. temperature:

```python
def aridity_threshold(t_annual, p_summer_fraction):
    """
    t_annual: mean annual temperature (°C)
    p_summer_fraction: fraction of precipitation in summer half-year
    Returns: threshold annual precipitation (mm)
    """
    if p_summer_fraction >= 0.7:
        return 20 * t_annual + 280   # summer-dominant rain
    elif p_summer_fraction <= 0.3:
        return 20 * t_annual          # winter-dominant rain
    else:
        return 20 * t_annual + 140   # even distribution
```

If annual precipitation < threshold → **B climate**.
If < threshold/2 → **BW (desert)**; otherwise **BS (steppe)**.

---

## Full Classification Algorithm

```python
def koppen_classify(monthly_temp, monthly_precip):
    """
    monthly_temp:   list of 12 monthly mean temperatures (°C)
    monthly_precip: list of 12 monthly precipitation values (mm)
    Returns: Köppen code string (e.g. "Cfa", "BWh", "Dfc")
    """
    t_ann = sum(monthly_temp) / 12.0
    p_ann = sum(monthly_precip)

    t_max = max(monthly_temp)    # warmest month temperature
    t_min = min(monthly_temp)    # coldest month temperature

    # Summer / winter half-years (N hemisphere convention; flip for S hemisphere)
    summer_months = [3,4,5,6,7,8]   # Apr–Sep
    winter_months = [9,10,11,0,1,2] # Oct–Mar
    p_summer = sum(monthly_precip[m] for m in summer_months)
    p_winter = sum(monthly_precip[m] for m in winter_months)
    p_summer_frac = p_summer / (p_ann + 1e-6)

    # Dry season detection
    p_min_summer = min(monthly_precip[m] for m in summer_months)
    p_min_winter = min(monthly_precip[m] for m in winter_months)
    p_max_summer = max(monthly_precip[m] for m in summer_months)
    p_max_winter = max(monthly_precip[m] for m in winter_months)

    # ── Group E (Polar) ──────────────────────────────────────
    if t_max < 10:
        return "ET" if t_max >= 0 else "EF"   # Tundra or Ice Cap

    # ── Group B (Arid) ───────────────────────────────────────
    threshold = aridity_threshold(t_ann, p_summer_frac)
    if p_ann < threshold:
        main = "BW" if p_ann < threshold / 2 else "BS"
        sub  = "h" if t_ann >= 18 else "k"    # hot vs. cold desert/steppe
        return main + sub

    # ── Groups A, C, D ───────────────────────────────────────

    # Determine precipitation sub-letter
    def precip_letter():
        # 'f' = no dry season
        # 's' = dry summer (Mediterranean)
        # 'w' = dry winter (monsoon / savanna)
        # 'm' = monsoon (A group special)

        # Dry summer: driest summer month < 40mm AND < p_max_winter/3
        dry_summer = p_min_summer < 40 and p_min_summer < p_max_winter / 3.0
        # Dry winter: driest winter month < p_max_summer/10
        dry_winter = p_min_winter < p_max_summer / 10.0

        if t_min >= 18:  # Group A
            if p_min < 60:
                # Monsoon if enough annual rain to compensate dry months
                if p_ann >= 25 * (100 - p_min):
                    return 'm'
                return 'w'
            return 'f'

        if dry_summer and not dry_winter:
            return 's'
        if dry_winter and not dry_summer:
            return 'w'
        return 'f'

    p_min = min(monthly_precip)

    # Group A (Tropical): all months > 18°C
    if t_min >= 18:
        return "A" + precip_letter()

    # Group C (Temperate): coldest month -3°C to 18°C
    if t_min >= -3:
        main = "C"
        prec = precip_letter()
        # Temperature sub-letter
        if t_max >= 22:
            sub = "a"   # hot summer
        elif sum(1 for t in monthly_temp if t >= 10) >= 4:
            sub = "b"   # warm summer
        else:
            sub = "c"   # cold summer
        return main + prec + sub

    # Group D (Continental): coldest month < -3°C
    main = "D"
    prec = precip_letter()
    if t_max >= 22:
        sub = "a"
    elif sum(1 for t in monthly_temp if t >= 10) >= 4:
        sub = "b"
    elif t_min < -38:
        sub = "d"   # extremely cold (Siberia)
    else:
        sub = "c"
    return main + prec + sub
```

---

## Common Köppen Codes

| Code | Description | Real-World Examples |
|---|---|---|
| Af | Tropical Rainforest | Amazon, Congo, SE Asia |
| Am | Tropical Monsoon | Bangladesh, parts of India |
| Aw | Tropical Savanna | Sahel, Brazilian Cerrado |
| BWh | Hot Desert | Sahara, Arabian Peninsula |
| BWk | Cold Desert | Gobi, Great Basin |
| Cfa | Humid Subtropical | SE USA, E China, Argentina |
| Cfb | Oceanic | W Europe, Pacific NW |
| Csb | Mediterranean (cool) | California, Portugal |
| Csa | Mediterranean (hot) | Greece, Lebanon |
| Dfb | Humid Continental | Midwest USA, Central Europe |
| Dfc | Subarctic | Canada, Siberia |
| ET  | Tundra | Arctic coasts, alpine |
| EF  | Ice Cap | Antarctica, Greenland |

---

## Generating Monthly Climate Data

Köppen requires monthly data, which must be synthesized from the physics simulation. Key drivers:

```
Monthly temperature estimation:
  t[month] = t_annual
            + amplitude * cos(2π * (month - t_peak) / 12)
            - lapse_rate * elevation_m / 1000

  Where:
    amplitude = f(latitude, continentality)
    t_peak    = 7 (July in N hemisphere; 1 in S hemisphere)
    continentality = f(distance from ocean)  — inland = larger swings

Monthly precipitation estimation:
  p[month] = ITCZ/monsoon position + orographic lift + prevailing wind

  (Full climate simulation required for accurate monthly precip)
```

---

## References

- Wikipedia — Köppen climate classification: https://en.wikipedia.org/wiki/K%C3%B6ppen_climate_classification
- Kottek et al. (2006) updated digital map: https://link.springer.com/article/10.1127/0941-2948/2006/0130
- WorldClim dataset (real-world climate data for validation): https://www.worldclim.org/
- Artifexian YouTube — How to create a realistic climate: https://www.youtube.com/watch?v=5lCbxMZJ4zA

---

## Application in Our World-Gen Pipeline

Köppen runs **after the full climate simulation** produces per-cell monthly temperature and precipitation. It is the most scientifically accurate biome classifier we can use.

| Output | Used By |
|---|---|
| **Climate zone map** | Biome assignment, rendering |
| **Aridity classification** | Desert type, water availability for civilizations |
| **Seasonal patterns** | Agriculture calendar, seasonal event simulation |
| **Monsoon regions** | River flood events, wet/dry season cycles |
| **Continentality map** | Temperature extremes → impact on civilization survival |

**Pipeline position:**
1. Tectonics → elevation
2. Erosion → refined elevation + rivers
3. Atmospheric/ocean circulation → wind + ocean currents
4. **Climate sim** → monthly temp/precip per cell
5. **Köppen** → climate zone label per cell
6. Whittaker or custom → biome label per cell
7. Ecology → vegetation density, species

Köppen is preferred over Whittaker for the final simulation; Whittaker is fine for a quick prototype pass.
