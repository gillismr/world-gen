# Agent-Based Modeling (ABM)

## Overview

Agent-Based Modeling (ABM) simulates systems by defining **autonomous agents** — entities with internal state and behavioral rules — and letting them interact with each other and their environment. Emergent phenomena (cities, civilizations, trade networks, wars, cultural diffusion) arise from the bottom-up interactions of many simple agents, rather than being prescribed top-down.

In procedural world generation, ABM is the primary technique for the **civilization and history simulation layer**: simulating the emergence of cultures, settlements, territories, wars, and dynasties over hundreds or thousands of years.

---

## Core Concepts

### Agent
An entity with:
- **State**: position, population, resources, culture, allegiance, etc.
- **Rules / behaviors**: how it acts based on its state and environment
- **Perception**: what it can sense (local neighborhood, global map)

### Environment
- The spatial world (map cells, graph nodes)
- Contains resources, terrain, climate data
- Shared between agents

### Interaction
Agents interact with:
- The environment (consume resources, modify terrain)
- Other agents (trade, war, cultural exchange, migration)

### Time
Discrete timesteps (turns). Each step, every agent evaluates its rules and acts.

---

## Civilization ABM — Architecture

```python
from dataclasses import dataclass, field
from typing import List, Dict, Optional
import random

@dataclass
class Culture:
    name: str
    language_family: str
    religion: str
    aggression: float = 0.5   # [0,1]
    expansion_drive: float = 0.5

@dataclass
class Settlement:
    id: int
    x: int
    y: int
    population: int
    food_stores: float
    culture: Culture
    faction_id: int
    is_capital: bool = False

@dataclass
class Faction:
    id: int
    name: str
    culture: Culture
    settlements: List[int] = field(default_factory=list)
    treasury: float = 0.0
    military_strength: float = 0.0
    age: int = 0

class HistorySimulation:
    def __init__(self, world_map, initial_cultures):
        self.world = world_map          # 2D grid with terrain/resources
        self.settlements: Dict[int, Settlement] = {}
        self.factions: Dict[int, Faction] = {}
        self.year = 0
        self.history_log = []

        # Seed initial settlements from cultural hearths
        self.seed_initial_settlements(initial_cultures)

    def seed_initial_settlements(self, cultures):
        """Place founding settlements in fertile, coastal, or river areas."""
        for culture in cultures:
            # Find a good location: fertile, near water, not too cold
            loc = self.find_settlement_site(culture)
            if loc:
                x, y = loc
                faction = Faction(id=len(self.factions),
                                  name=generate_faction_name(culture),
                                  culture=culture)
                settlement = Settlement(id=len(self.settlements),
                                        x=x, y=y,
                                        population=1000,
                                        food_stores=500.0,
                                        culture=culture,
                                        faction_id=faction.id,
                                        is_capital=True)
                faction.settlements.append(settlement.id)
                self.settlements[settlement.id] = settlement
                self.factions[faction.id] = faction

    def step(self):
        """Advance one year of history."""
        self.year += 1

        # Each settlement acts
        for sid, settlement in list(self.settlements.items()):
            self.settlement_step(settlement)

        # Each faction makes strategic decisions
        for fid, faction in list(self.factions.items()):
            self.faction_step(faction)

        # Global events (plague, volcanic eruption, etc.)
        self.random_events()

    def settlement_step(self, s: Settlement):
        """Per-turn logic for a single settlement."""
        # 1. Harvest food from surrounding cells
        fertility = self.world.fertility(s.x, s.y)
        food_production = fertility * s.population * 0.01
        s.food_stores += food_production

        # 2. Consume food (population needs 1 unit/person/year)
        s.food_stores -= s.population

        # 3. Population growth/decline based on food surplus
        if s.food_stores > 0:
            growth_rate = 0.02 * min(1.0, s.food_stores / s.population)
            s.population = int(s.population * (1 + growth_rate))
        else:
            # Famine
            s.population = int(s.population * 0.95)
            self.log_event("famine", s)

        # 4. Expansion: if overpopulated, found a new settlement or conquer
        if s.population > self.overpopulation_threshold(s):
            self.try_expand(s)

    def try_expand(self, s: Settlement):
        """Attempt to found a new settlement or conquer a neighbor."""
        faction = self.factions[s.faction_id]

        # Find adjacent empty, fertile cells
        empty_sites = self.find_empty_sites_near(s.x, s.y, radius=5)

        if empty_sites and random.random() < 0.3:
            # Found new settlement
            site = random.choice(empty_sites)
            new_s = Settlement(id=len(self.settlements),
                               x=site[0], y=site[1],
                               population=s.population // 4,
                               food_stores=100.0,
                               culture=s.culture,
                               faction_id=s.faction_id)
            s.population -= new_s.population
            self.settlements[new_s.id] = new_s
            faction.settlements.append(new_s.id)
            self.log_event("found_settlement", new_s, parent=s)
        else:
            # Consider war with nearest rival faction
            rival = self.nearest_rival(s)
            if rival and faction.culture.aggression > 0.5:
                self.declare_war(faction, self.factions[rival.faction_id])

    def faction_step(self, faction: Faction):
        """Faction-level decisions: diplomacy, war, trade."""
        faction.age += 1
        faction.military_strength = sum(
            self.settlements[sid].population * 0.05
            for sid in faction.settlements
            if sid in self.settlements
        )

        # Check for faction collapse (all settlements lost)
        if not faction.settlements:
            self.log_event("faction_collapse", faction)

    def random_events(self):
        """Low-probability world events."""
        if random.random() < 0.001:   # 0.1% per year
            self.trigger_plague()
        if random.random() < 0.0005:
            self.trigger_volcanic_eruption()

    def log_event(self, event_type, *args, **kwargs):
        self.history_log.append({
            'year': self.year,
            'type': event_type,
            'args': args,
            'kwargs': kwargs
        })
```

---

## Cultural Diffusion

Culture spreads between adjacent settlements based on contact frequency and relative power:

```python
def cultural_diffusion_step(settlements, factions, diffusion_rate=0.01):
    for s in settlements.values():
        neighbors = find_neighboring_settlements(s, settlements, radius=3)
        for n in neighbors:
            # Stronger/larger neighbor exerts more cultural influence
            influence = n.population / (s.population + n.population)
            if random.random() < diffusion_rate * influence:
                # Gradual cultural shift (religion, language, etc.)
                s.culture = blend_cultures(s.culture, n.culture, influence * 0.1)
```

---

## Mesa Framework (Python)

Mesa is the standard Python ABM framework. It handles agent scheduling, spatial grids, and visualization:

```python
from mesa import Agent, Model
from mesa.space import MultiGrid
from mesa.time import RandomActivation
from mesa.datacollection import DataCollector

class CivAgent(Agent):
    def __init__(self, unique_id, model, culture):
        super().__init__(unique_id, model)
        self.culture = culture
        self.population = 1000

    def step(self):
        self.move()
        self.grow()
        self.interact()

class WorldModel(Model):
    def __init__(self, width, height, n_agents):
        self.grid = MultiGrid(width, height, torus=True)
        self.schedule = RandomActivation(self)
        for i in range(n_agents):
            agent = CivAgent(i, self, random_culture())
            self.schedule.add(agent)
            self.grid.place_agent(agent, (random.randrange(width),
                                          random.randrange(height)))

    def step(self):
        self.schedule.step()
```

---

## References

- Wikipedia — Agent-based model: https://en.wikipedia.org/wiki/Agent-based_model
- Mesa (Python ABM framework): https://mesa.readthedocs.io/
- NetLogo models library: https://ccl.northwestern.edu/netlogo/models/
- MASON framework (Java): https://cs.gmu.edu/~eclab/projects/mason/
- Epstein & Axtell "Growing Artificial Societies" (1996) — foundational text
- Schelling model (segregation): https://en.wikipedia.org/wiki/Schelling%27s_model_of_segregation
- "civs" (companion to WorldEngine): https://github.com/Mindwerks/civs

---

## Application in Our World-Gen Pipeline

ABM is the engine for the **civilization and history layers** — the most complex parts of the pipeline.

| Agent Type | Simulation Layer | Represents |
|---|---|---|
| Settlement agent | Civilization | Town/city with population, food, culture |
| Faction/nation agent | Civilization | Political entity controlling settlements |
| Person / leader agent | History | Named historical figures (kings, generals, heroes) |
| Trade caravan | Civilization | Mobile trade route activity |
| Army | History | Military campaign with objectives |
| Religion | Culture | Spreading belief system with conversion rules |

**Emergent phenomena we want:**
- Capitals naturally form at resource-rich, defensible locations
- Trade networks follow river valleys and mountain passes
- Empires rise and fall based on military overextension and resource stress
- Cultural diffusion produces realistic linguistic and religious boundaries
- Historical events (plagues, volcanic eruptions) cascade into political collapse

**Key design decision:** Do we simulate **individual people** (like Dwarf Fortress) or **aggregate populations** (like Civilization)? Individual simulation produces richer history but scales poorly. Population-level simulation is faster and scales to thousands of years. A hybrid (aggregate normally, switch to individual for named historical figures) is worth exploring.
