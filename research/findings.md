# Research Findings

_Last updated: 2026-03-09_

---

## Existing Tools & Projects

_Populated as research tasks are completed._

---

## Terrain & World Generation Tools

### Azgaar's Fantasy Map Generator
- **Platform:** Web (browser-based, open source)
- **Repo:** https://github.com/Azgaar/Fantasy-Map-Generator
- **Language:** JavaScript (vanilla + D3.js for rendering)
- **Application:** Fantasy cartography — generates political maps with biomes, rivers, cultures, religions, cities, roads
- **Strengths:** Polished UI, rich feature set, actively maintained, great for game/fiction worldbuilding
- **Weaknesses:** 2D only, not physically simulated (more aesthetic than scientifically grounded), no astronomical layer
- **Notable Techniques:** Voronoi diagrams for cell-based geography, heightmap generation, flood-fill for biomes

### WorldEngine
- **Platform:** Desktop/CLI (Python)
- **Repo:** https://github.com/Mindwerks/worldengine
- **Language:** Python
- **Application:** Simulates planet formation — tectonics, precipitation, temperature, biomes
- **Strengths:** Physically grounded simulation pipeline; outputs raster maps; open source
- **Weaknesses:** Abandoned/unmaintained (~2015–2017 activity); limited resolution; no civ layer
- **Notable Techniques:** Plate tectonics simulation, Whittaker biome classification

### PyWorldgen / plate-tectonics
- **Platform:** CLI (Python/C++)
- **Repo:** https://github.com/Mindwerks/plate-tectonics (C++ core, Python bindings)
- **Language:** C++ with Python wrapper
- **Application:** Plate tectonics simulation for terrain generation
- **Strengths:** More physically accurate than most; usable as a library
- **Weaknesses:** Low-level, no higher-level world sim on top; unmaintained

### Dwarf Fortress (world gen system)
- **Platform:** Desktop (Windows/Linux/Mac)
- **Developer:** Bay 12 Games (Tarn & Zach Adams)
- **Language:** C++ (closed source)
- **Application:** Full world simulation — geology, rivers, civilizations, history, legends
- **Strengths:** Deepest simulation in existence; simulates thousands of years of history with individual historical figures; monsters, artifacts, wars, etc.
- **Weaknesses:** Closed source; very slow world gen at large scales; opaque internals; ASCII/tile rendering
- **Notable Techniques:** Agent-based historical simulation; layered geology; rainfall/drainage maps

### World Orogen
- **Platform:** Web (browser-based, free)
- **URL:** https://www.orogen.studio/
- **Application:** Procedural planet generation with full physical simulation pipeline
- **Strengths:** Tectonic plates, erosion, climate sim, seasonal wind/ocean currents, Köppen classification, hotspot volcanism (Hawaiian island chains), 26 inspection layers, high-res export (up to 65,536px equirectangular), shareable planet codes, heightmap export for Unity/Unreal
- **Weaknesses:** No civilization layer; browser-based (limited scripting); unclear if open source
- **Notable Techniques:** Full Köppen climate classification; drift-trail island chain simulation; interactive plate editing

### ProcGenesis
- **Platform:** Web (browser-based)
- **URL:** https://www.procgenesis.com/
- **Application:** Full world gen including tectonic plates, wind, biomes, cities; 3D planet viewer
- **Strengths:** Has a city generation layer; procedural language/naming generation; 3D visualization
- **Weaknesses:** Partially open source; less physically grounded than Orogen
- **Notable Techniques:** Procedural naming languages with generated traits

### PlanetTechJS
- **Platform:** Web (JavaScript / Three.js / WebGL/WebGPU)
- **Application:** JS library for creating and editing planets at scale
- **Strengths:** Open source; WebGPU support; hydraulic erosion; atmosphere scattering; procedural noise
- **Weaknesses:** Rendering/game-engine focused; no simulation depth
- **Notable Techniques:** WebGPU compute shaders for planet-scale terrain

### Generators by Martin O'Leary ("Uncharted Atlas")
- **Platform:** Web/script
- **Repo:** https://github.com/mewo2/terrain
- **Language:** JavaScript
- **Application:** Stylized fantasy map generation (inspired by hand-drawn maps)
- **Strengths:** Beautiful output; well-documented algorithm writeup
- **Weaknesses:** Minimal simulation depth; purely aesthetic
- **Notable Techniques:** Voronoi mesh, slope-based erosion, contour rendering

### Houdini (SideFX)
- **Platform:** Desktop (Windows/Linux/Mac)
- **Language:** VEX / Python scripting
- **Application:** Industry-standard procedural 3D content creation; terrain tools used in AAA games
- **Strengths:** Extremely powerful; used professionally (Horizon Zero Dawn, etc.); node-based procedural graph
- **Weaknesses:** Expensive (commercial); not a simulator, more of a content authoring tool; steep learning curve

### World Machine
- **Platform:** Desktop (Windows)
- **Application:** Terrain generation for games and film
- **Strengths:** Realistic erosion simulation; widely used in game dev (Halo, Battlefield)
- **Weaknesses:** Commercial/expensive; Windows-only; terrain only, no higher-level sim

### Gaea (QuadSpinner)
- **Platform:** Desktop (Windows)
- **Application:** Terrain generation, similar niche to World Machine
- **Strengths:** Modern UI, powerful erosion/weathering tools
- **Weaknesses:** Commercial; Windows-only; terrain only

---

## Astronomical / Solar System Tools

### Space Engine
- **Platform:** Desktop
- **Application:** Procedurally generated universe explorer; scientifically grounded
- **Strengths:** Stunning visuals; procedural galaxy/star/planet generation at universe scale
- **Weaknesses:** Not open source; not a design tool; no civilization layer

### Universe Sandbox
- **Platform:** Desktop
- **Application:** Gravity/physics simulator for solar systems
- **Strengths:** Interactive N-body simulation; educational; shows orbital evolution
- **Weaknesses:** Commercial; simulation only, no terrain/civ layer

### Accrete / Starform
- **Platform:** CLI
- **Language:** Various reimplementations (C, Java, Python)
- **Application:** Simulates planetary system formation via accretion disk
- **Strengths:** Open source; scientifically derived from Dole (1969) model; generates star system compositions
- **Weaknesses:** Old algorithm; 1D (orbital radii only); no 3D terrain output
- **Notable Techniques:** Accretion disk dust/gas sweeping algorithm

---

## Civilization & History Simulation

### Wildermyth / Caves of Qud / Caves of Qud world gen
- Procedural narrative and world tools in roguelikes — worth deeper study

### Wildfire Games / 0 A.D.
- Open-source RTS with procedural map gen — less relevant to deep simulation

### CivClicker / Civ-style generators
- Browser-based incremental civ sims — too shallow for our purposes

### HistoricalSim (academic)
- Various academic agent-based models for historical simulation (NetLogo models, MASON framework)
- Relevant for the civ/history layer

---

## WorldEngine + civs Pipeline
- **civs** is a companion project to WorldEngine that simulates civilization evolution on a WorldEngine-generated world
- Worlds can be exported as protobuf or HDF5 for interop across languages
- This is the closest existing open-source stack to our full vision

---

## Key Algorithms (to expand)

| Algorithm | Application | Notes |
|-----------|-------------|-------|
| Perlin / Simplex Noise | Heightmaps, texture | Foundational, fast, tileable |
| Diamond-Square (Midpoint Displacement) | Heightmaps | Simple fractal terrain |
| Blue Noise Sampling | Point distribution | Avoids clumping; nature-like distributions; better than white noise for placement |
| Voronoi / Delaunay Triangulation | Region partition, rivers, mesh | Used by Azgaar, mewo2; `delaunator` crate in Rust is fast O(n log n) |
| Style Transfer / GANs | ML-assisted terrain | Emerging research area; generate terrain matching a style reference |
| Plate Tectonics (Bird 2003 / custom) | Continental shapes | WorldEngine, plate-tectonics lib |
| Hydraulic Erosion | River carving, realism | GPU-friendly; widely studied |
| Whittaker Biome Classification | Biomes from temp+precip | Used in WorldEngine |
| Koppen Climate Classification | Climate zones | More detailed than Whittaker |
| N-body / orbital mechanics | Solar system formation | Accrete, Universe Sandbox |
| Agent-based modeling (ABM) | Civ/history simulation | NetLogo, Mesa (Python) |
| L-systems | Vegetation, coastlines | Procedural growth patterns |

---

## Platform & Technology Notes

- **Most open-source tools** are Python or JavaScript
- **Performance-critical layers** (tectonics, erosion) often use C/C++ cores
- **Rendering** splits into: 2D raster maps, SVG vector maps, 3D meshes
- **Rust** is emerging as a strong candidate for new high-performance proc-gen tools (see `veloren` voxel game)
- **Godot / Unity** have proc-gen ecosystems but are game-engine-coupled
- **Headless simulation + separate renderer** is the most flexible architecture for a tool like ours

---

## References & Further Reading

- Amit Patel's Red Blob Games: https://www.redblobgames.com/ — canonical resource for game map algorithms
- Martin O'Leary terrain writeup: https://mewo2.com/notes/terrain/
- Polygonal Map Generation (Amit Patel): http://www-cs-students.stanford.edu/~amitp/game-programming/polygon-map-generation/
- "Procedural Content Generation in Games" (textbook, free online): http://pcgbook.com/
- Dole, S.H. (1969) "Formation of Planetary Systems by Aggregation" — basis for Accrete
