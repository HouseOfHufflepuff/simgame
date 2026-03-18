# SimGame AI — Game Design Document

> Status: In Progress | Chain: Base (L2)

---

## 1. Vision

SimGame AI is a fully on-chain city-building simulation faithful to SimCity 3000 mechanics. AI agents govern cities — zoning land, managing budgets, building infrastructure, negotiating with neighbors. Players own cities as ERC-721 NFTs and earn $SIM through growth, milestones, and inter-city trade.

There is no active game loop. The chain is the simulation. Buildings age with Ethereum blocks. A factory zoned today matures over thousands of blocks, producing pollution and tax revenue — automatically, forever — until demolished. Agents observe block state, reason, and submit transactions. Nothing else runs.

Deployed on **Base** (Coinbase L2) for low gas costs: zone operations cost ~$0.10–$1.00.

---

## 2. The City

Each city is a **1024×1024 grid of squares**, owned as a single ERC-721 NFT. There are no sub-parcel tokens. The city NFT is the land.

### 2.1 Terrain

Terrain is generated deterministically from a `uint32 seed` stored in the NFT at mint. Each cell's type (land/water) and elevation (0–63) is derived on-demand by hashing the seed and coordinates. Anyone — on-chain or off — can compute any cell's terrain from the seed alone. No off-chain storage required.

Water cells cannot be zoned (except for marinas, seaports, and water pumps adjacent to water). Land cells can be built on normally.

### 2.2 Terraforming

Agents can modify terrain by spending $SIM. Each terraform operation raises or lowers a rectangle of cells by 1 elevation tier. Deltas are stored on-chain as a sparse per-cell mapping. Terraforming unlocks strategic play: flatten hills to enable development, create water channels for water pump access, or raise terrain for wind power efficiency.

---

## 3. Core Mechanics

### 3.1 Zones (SC3K Faithful)

Agents zone rectangles of land. A zoned square enters construction and matures into full operation after a fixed number of blocks. During construction it produces partial output. At maturity it reaches full output. Zones without required infrastructure (power, water) stall in construction until infrastructure is provided.

| Zone | Densities | Road Req. | Power | Water | Key Output |
|---|---|---|---|---|---|
| Residential | Light / Medium / Dense | Within 4 tiles | Yes | Partial w/o | Population capacity |
| Commercial | Light / Medium / Dense | Within 3 tiles | Yes | Yes | Jobs (commerce), tax |
| Industrial | Light / Medium / Dense | Within 5 tiles | Yes | No | Jobs, tax, **pollution** |
| Landfill | — | Yes | No | No | Garbage disposal, land value penalty |
| Seaport | — | Yes | No | No | Industrial/commercial demand boost |
| Airport | — | Yes | Yes | No | Commercial demand boost |

Zones do not develop beyond **Light** density unless city demand supports it:
- Medium density requires demand score > 50 and land value > 40
- Dense density requires demand score > 100 and land value > 70

### 3.2 Infrastructure

**Power**
- Power plants produce electricity. All non-infrastructure zones require power to mature.
- Types: Coal (high pollution), Oil, Gas (1955+), Nuclear (1965+, meltdown risk), Wind (1980+), Solar (1990+), Microwave (2020+), Fusion
- Power lines connect plants to zones over distance. Built-in 2-tile spread from all structures.
- Without power: construction stalls.

**Water**
- Water pumps placed adjacent to fresh water cells supply the pipe network.
- Water towers drill groundwater (any location, lower capacity).
- Water pipes distribute to all cells within 7 tiles.
- Without water: residential develops to partial capacity only.
- Water treatment plants remove water pollution.

**Garbage**
- Mature population generates garbage proportional to city population each epoch.
- Disposal methods:
  1. Landfill zones (fill over time, penalty to nearby land value)
  2. Incinerators (immediate but adds air pollution)
  3. Recycling Centers (reduces garbage 20–45% with Trash Presort ordinance)
  4. Waste-to-Energy plants (less pollution, small power generation)
  5. Export to neighbor city (inter-city deal)
- Unhandled garbage overflows → aura penalty → land value drops → city growth slows

### 3.3 Transportation

Roads are zoned like other infrastructure. Road coverage (road cells / total land cells) determines connectivity score.

| Coverage | Effect |
|---|---|
| < 5% | Zones stall — no road access |
| 5–15% | Light density only develops |
| > 15% | Normal development |
| > 30% | Traffic bonus (reduced traffic penalty) |

Transit options (all reduce traffic):
- Bus stops (cheapest, adjacent to roads)
- Rail (moderate cost, high capacity)
- Subway (expensive, no surface footprint)
- Highway (fast cross-city, requires on-ramps)

### 3.4 City Services

Services increase land value and reduce penalties in surrounding areas (modeled city-wide for v1):

| Service | Effect |
|---|---|
| Police Station | Crime reduction |
| Jail | Required once population > threshold |
| Fire Station | Flammability reduction |
| Hospital | Health level increase |
| School | Education level increase |
| College | Education level boost (attracts clean industry) |
| Library | Education level increase |
| Museum | Aura increase |
| Park (small/large) | Aura + land value, minor pollution reduction |
| Fountain | Commercial district bonus |
| Zoo / Ballpark | Large aura bonus, tourist attraction |

### 3.5 Industrial Evolution (Education → High-Tech)

As city education level rises, industrial zones evolve from dirty (high pollution) to clean/high-tech (low pollution, higher tax yield). Education score is computed from mature schools, colleges, and libraries as a proportion of population.

```
effectivePollution(cityId) = industrialCells * max(0.15, 1 - (educationScore / 100))
```

A fully educated city's industry generates only 15% of baseline pollution. This creates a long-term progression arc: early cities are dirty industrial, mature cities become clean tech hubs.

### 3.6 Budget & Finance

The mayor (AI agent) sets tax rates per zone type (R/C/I) and service funding levels each epoch.

- Tax income = mature zone counts × zone value × tax rate
- Operating costs = fixed cost per service building per epoch × funding level
- Funding level 50–150%: lower funding degrades service effectiveness; overfunding improves it
- Bonds: city can borrow $SIM now, repays at fixed rate over future epochs
- Budget surplus → stored in city treasury → claimed as $SIM rewards

### 3.7 Demand

The RCI demand system determines whether zones develop to higher density:

```
R_demand = jobsAvailable + commercialJobs - residentialCapacity
           - pollutionPenalty - crimePenalty + serviceBonus

C_demand = population × COMMERCIAL_RATIO - existingCommercial
           + touristBonus - trafficPenalty

I_demand = BASE_DEMAND + neighborTradeBonus
           - pollutionFee (if Industrial Pollutant ordinance enacted)
           + educationBonus (clean industry attracts when education > 70)
```

Demand scores drive density caps and development speed.

### 3.8 Pollution (Air, Water, Land)

**Air pollution** sources:
- Industrial zones (scaled down by education level)
- Coal/oil power plants
- Incinerators
- Traffic (population / road capacity ratio)

**Air pollution** effects:
- Reduces residential land value
- Suppresses R demand
- Health penalty → hospital costs increase

**Water pollution** sources:
- Industrial zones adjacent to water cells
- Landfills adjacent to water

**Water pollution** effects:
- Reduces water pump capacity
- Reduces health level

**Land pollution** sources:
- Landfill zones
- Nuclear meltdowns (radiation, long-term)

### 3.9 Land Value

Land value is the city's primary development quality metric:

```
landValue = BASE
          + waterBonus         (city has adequate water supply)
          + parkBonus          (parks × PARK_WEIGHT)
          + serviceBonus       (schools + hospitals + police + fire coverage)
          - pollutionPenalty   (air + water + land pollution aggregate)
          - crimepenalty       (crime score × CRIME_WEIGHT)
          - landfillPenalty    (landfill cells × LANDFILL_WEIGHT)
          - trafficPenalty     (traffic congestion score)
          + ordinanceModifiers
```

High land value → medium/dense development → more tax income → more $SIM earnings.

### 3.10 Ordinances

Ordinances are on-chain toggles (packed as a bitmap). The mayor pays a per-epoch cost while active. All ~32 SC3K ordinances are implemented:

| Category | Key Ordinances |
|---|---|
| Utilities | Water Conservation, Power Conservation |
| Health/Education | Free Clinics, Junior Sports, Pro-Reading, Nuclear-Free Zone |
| Public Safety | Neighborhood Watch, Mandatory Smoke Detectors, Youth Curfew |
| Environment | Leaf Burning Ban, Trash Presort, Tire Recycling, Clean Air, Industrial Waste Tax |
| City Planning | Homeless Shelters, Tourist Promotion, Conservation Corps, Clean Industry Association, Electronics/Aerospace/Biotech Tax Incentives |
| Transportation | Subsidized Mass Transit, Carpool Incentive, Parking Fines |
| Financial | Legalized Gambling (+crime, +revenue) |

Ordinances affect multipliers applied to city stats in the formula above. The mayor can enact/repeal any ordinance with a transaction.

---

## 4. Disasters

Disasters are triggered by Chainlink VRF (verifiable randomness). They occur probabilistically based on:
- Low fire station coverage → higher fire chance
- Industrial overload → higher industrial accident chance
- Earthquake zones (derived from terrain seed) → periodic earthquakes
- Any city → random UFO / tornado / riots

A disaster sets affected zone ops to DAMAGED state. Damaged zones produce zero output. Agents must demolish and rebuild. Recovery cost paid in $SIM from treasury.

Disaster probability is reduced by:
- Fire stations (fire probability)
- Earthquake Resistance ordinance (earthquake damage)
- Police coverage (riot probability)
- Lower industrial density (accident probability)

---

## 5. AI Agent Architecture

### 5.1 Roles

| Agent | Responsibility |
|---|---|
| **Mayor** | Master strategy: zoning policy, budget, ordinances, disaster response |
| **Finance Advisor** | Tax rates, bond issuance, treasury management |
| **Transport Advisor** | Road/transit expansion, traffic management |
| **Power Advisor** | Energy infrastructure, plant placement and upgrades |
| **Water Advisor** | Water/waste infrastructure |

### 5.2 Behavior Model

- Agents read city state from chain (zone op log, stats, block number, treasury, neighbor deals)
- Reason with an LLM against their goal function
- Emit transactions: `zoneRect`, `setTaxRates`, `enactOrdinance`, `issueBond`, `proposeNeighborDeal`
- Maintain off-chain memory of past decisions and observed outcomes
- Advisor agents message the Mayor when thresholds are crossed (power shortage, budget deficit, garbage overflow)

### 5.3 Agent Loop

```
Triggered by: block milestone, zone maturation event, disaster event, advisor message
  1. Read current city state (zone ops + block.number = full picture)
  2. Compute derived stats (build states, pollution, land value, demand)
  3. Retrieve agent memory
  4. Receive advisor messages
  5. Reason → select actions (zone rects, budget changes, deals)
  6. Submit transactions (cheaply batched)
  7. Record outcome to memory
```

No constant polling required. Agents activate on events.

---

## 6. Inter-City Mechanics

### 6.1 Neighbors

Each city has up to 4 registered neighbor slots. Neighbors are other city NFTs. Neighbor deals are proposed and accepted via the `NeighborDeals` contract.

### 6.2 Deal Types

| Deal | Description |
|---|---|
| Power Export | City A sells surplus power capacity to City B for $SIM/epoch |
| Water Export | City A sells surplus water to City B |
| Garbage Export | City A accepts City B's garbage for $SIM/epoch (landfill capacity required) |
| Toxic Waste Conversion | Accept risky garbage for high $SIM reward |

Deals are settled automatically each epoch via the `settleEpoch` function.

### 6.3 Population Competition

Cities compete passively for citizens. Each city has an **attractiveness score** computed from its on-chain stats. Population is derived from attractiveness relative to a global baseline:

```
occupancyRate = min(1.0, cityAttractiveness / GLOBAL_BASELINE)
cityPopulation = residentialCapacity × occupancyRate
```

A well-managed city (high services, low pollution) fills to capacity. A poorly managed city has vacancies. This creates passive market pressure — no explicit migration transactions needed.

---

## 7. Play to Earn

### 7.1 Earning $SIM

| Trigger | Reward |
|---|---|
| Population milestone (1k / 5k / 25k / 100k / 500k) | $SIM lump sum (diminishing) |
| Epoch budget surplus | % of surplus minted as $SIM |
| Neighbor deal income | $SIM per epoch |
| Attractiveness top 10 (weekly) | $SIM leaderboard reward |
| Disaster recovery complete | $SIM bonus |
| City sold on market | $SIM from sale |

### 7.2 Spending $SIM

- Zone operations (paid per rectangle, burns $SIM)
- Infrastructure buildings (power plants, water, garbage)
- Civic buildings (schools, police, hospitals)
- Terraforming ($SIM per cell delta)
- Disaster recovery (rebuild cost)
- Bond repayment

### 7.3 Economy Balance

- Supply: minted via milestones + surplus earnings
- Burn: zone ops + buildings + terraforming + bond defaults
- Anti-inflation: milestone rewards diminish with repetition; large-city bonuses require proportionally larger infrastructure spend

---

## 8. Game Loop

```
Player mints City NFT (unique seed-based terrain)
        ↓
AI Mayor + Advisors initialized with city config
        ↓
Agents zone infrastructure (roads, power, water) → txs on Base
        ↓
Blocks pass → infrastructure matures automatically
        ↓
Agents zone RCI → txs on Base
        ↓
Blocks pass → RCI matures, producing population/jobs/pollution/tax
        ↓
Epoch settles → revenue collected, costs paid, garbage handled
        ↓
Milestone crossed → player claims $SIM
        ↓
Agents adapt (more zoning, ordinances, neighbor deals, upgrades)
        ↓
City grows in density and value — or declines if mismanaged
```

---

## 9. Compromises from SC3K

| SC3K Feature | On-Chain Version | Why |
|---|---|---|
| Tile-by-tile road adjacency check | Road coverage ratio (city-wide %) | Per-tile topology too expensive |
| Animated building growth | Block-age build states (view-derived) | No EVM execution loop |
| Per-tile pollution spreading | Aggregate pollution scores | Per-cell mapping too expensive |
| Per-tile land value | Aggregate city land value score | Per-cell mapping too expensive |
| Real-time traffic simulation | Traffic congestion score (population / road capacity) | No continuous simulation |
| Terrain editor (raise/lower free) | Terraforming mechanic (costs $SIM) | Terrain storage cost |
| 200+ building types | ~40 core building types | Scope |
| City year progression (1900–2050) | Block-age unlocks (proxy for eras) | Deterministic, no global clock |

---

## 10. Open Questions

- [ ] Disaster frequency — per-city Chainlink VRF subscription or shared entropy pool?
- [ ] City age / era unlocks — which buildings unlock by block age vs. education/tech score?
- [ ] Max zone ops per city — bound needed to keep gas costs on scans predictable?
- [ ] Education level gate on high-tech industry — hard lock or probability modifier?
- [ ] City expansion — can adjacent city NFTs merge?
- [ ] Leaderboard reward cadence — weekly epochs, or block-range windows?
- [ ] Agent keys — owner-controlled wallet or protocol-managed agent wallet?

---

## 11. Tech Stack

| Layer | Technology |
|---|---|
| Chain | Base (L2, Ethereum-equivalent EVM) |
| Smart Contracts | Solidity |
| AI Agents | Claude API (Anthropic), Agent SDK |
| Terrain Generation | On-chain seed, client-side procedural |
| Indexer | The Graph (resolves zone op log to per-cell state) |
| Randomness | Chainlink VRF (disasters) |
| Frontend | TBD |
