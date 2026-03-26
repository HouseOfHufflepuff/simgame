# LunarSim — Technical Design

> Chain: Base (L2) | Status: In Progress

---

## The Idea in Plain Terms

Wallets own colonies. Colonies are placed on a shared lunar surface map. Inside each colony, players (or AI agents) excavate underground and place modules — habitation pods, mining rigs, life support systems, research labs, greenhouses. Modules age with Ethereum blocks. As they mature they produce income. Players claim that income. Colonies on the surface map compete for settlers and can trade resources with neighbors.

No simulation loop. No server. The chain is the clock.

---

## 1. The Surface Map

The lunar surface is a coordinate grid. Each colony occupies one tile on that grid. When you mint a colony you pick an empty surface tile and place it there. Some tiles are ice deposits — they cannot be built on but provide permanent water and oxygen bonuses to adjacent colony tiles.

Colonies on adjacent tiles (up, down, left, right) are **neighbors**. Neighbors can make trade deals and compete for population.

### 1.1 Storage

```solidity
// Underlying mapping is unbounded — any uint16 coordinate is writable.
// Validity (bounds + ice check) is enforced at mint, not at the mapping level.
mapping(uint16 surfaceX => mapping(uint16 surfaceY => uint256 colonyId)) public surfaceMap;

// World seed: set once at deploy. Drives all ice deposit placement deterministically.
uint256 public immutable worldSeed;

// Active surface bounds — a square centered on SURFACE_CENTER.
// Only tiles within radius can be minted. Expands automatically (see §1.3).
uint16  public surfaceRadius;                    // half-side of current valid square
uint16  public constant SURFACE_CENTER = 32768;  // center of uint16 space
uint8   public expansionCount;                   // how many times the surface has expanded
uint256 public totalColonies;                    // incremented on every mint
```

Initial state at deploy: `surfaceRadius = 32`, giving a **64×64 active area** (4,096 tiles, ~3,200 buildable after ice deposits).

### 1.2 Ice Deposits

Ice deposit tiles are computed on demand from `worldSeed` — no storage, no admin input. They form natural clusters (ice fields, polar caps) via a two-pass check: a tile is an ice deposit if it hashes as a core ice tile, or if any of its four neighbors does.

```solidity
uint8 public constant ICE_CORE_PCT = 8; // ~8% are "core" ice → ~22% total after clustering

function isIceDeposit(uint16 x, uint16 y) public view returns (bool) {
    // A tile is ice if it OR any cardinal neighbor is a core ice tile.
    // This creates natural ice fields and polar cap formations without storing any map data.
    return _isCore(x, y)
        || _isCore(x,     uint16(y - 1))
        || _isCore(x,     uint16(y + 1))
        || _isCore(uint16(x - 1), y)
        || _isCore(uint16(x + 1), y);
}

function _isCore(uint16 x, uint16 y) internal view returns (bool) {
    return uint256(keccak256(abi.encodePacked(worldSeed, x, y))) % 100 < ICE_CORE_PCT;
}
```

`isIceDeposit` is a pure view — zero gas, callable by anyone. The UI reads it to render the surface map. The contract calls it at mint to reject ice tile selections.

**Why clustering matters:** with random scatter at 8%, ice tiles would be isolated dots. The neighbor-expansion rule groups them into ice fields and polar formations (~22% of all tiles end up as ice). This creates real geographic variety — ice-adjacent colonies with water/oxygen advantages vs. dry interior colonies that must import.

### 1.3 Auto-Expanding Bounds

The surface starts small and unlocks new rings of tiles as it fills up. Expansion is triggered by anyone once the fill threshold is met — it costs only the gas of one write.

```solidity
uint8 public constant EXPAND_FILL_PCT = 60; // 60% fill triggers next expansion

function canExpand() public view returns (bool) {
    uint256 side      = uint256(surfaceRadius) * 2;
    uint256 allTiles  = side * side;
    uint256 buildable = allTiles * 78 / 100; // ~78% non-ice
    return totalColonies * 100 >= buildable * EXPAND_FILL_PCT;
}

function expandSurface() external {
    require(canExpand(), "fill threshold not met");
    surfaceRadius   *= 2;
    expansionCount  += 1;
    emit SurfaceExpanded(surfaceRadius, totalColonies, block.number);
}
```

**Expansion schedule (approximate):**

| Expansion | Radius | Grid size | Buildable tiles | Triggered when |
|---|---|---|---|---|
| 0 (launch) | 32 | 64×64 | ~3,200 | — |
| 1 | 64 | 128×128 | ~12,800 | ~1,920 colonies minted |
| 2 | 128 | 256×256 | ~51,200 | ~7,680 colonies minted |
| 3 | 256 | 512×512 | ~204,800 | ~30,720 colonies minted |
| 4 | 512 | 1024×1024 | ~819,200 | ~122,880 colonies minted |

The surface never runs out of space — it just unlocks new territory as the game grows. Early colonies are in the dense core and compete intensely. Late colonies have more room but are farther from established trading partners.

### 1.4 Mint Validation

```solidity
function mintColony(uint16 surfaceX, uint16 surfaceY) external payable {
    // 1. Within active bounds
    int32 dx = int32(uint32(surfaceX)) - int32(uint32(SURFACE_CENTER));
    int32 dy = int32(uint32(surfaceY)) - int32(uint32(SURFACE_CENTER));
    require(
        dx >= -int32(uint32(surfaceRadius)) && dx < int32(uint32(surfaceRadius)) &&
        dy >= -int32(uint32(surfaceRadius)) && dy < int32(uint32(surfaceRadius)),
        "outside surface bounds"
    );

    // 2. Not an ice deposit
    require(!isIceDeposit(surfaceX, surfaceY), "cannot build on ice");

    // 3. Not already claimed
    require(surfaceMap[surfaceX][surfaceY] == 0, "tile occupied");

    // 4. Compute and store ice adjacency bonus
    uint8 adjIce = _countAdjacentIce(surfaceX, surfaceY);

    // 5. Mint NFT, store colony
    uint256 colonyId = _mintNFT(msg.sender);
    surfaceMap[surfaceX][surfaceY] = colonyId;
    colonies[colonyId] = Colony({
        surfaceX:          surfaceX,
        surfaceY:          surfaceY,
        adjacentIceTiles:  adjIce,
        foundedBlock:      uint32(block.number),
        // ... rest of defaults
    });
    totalColonies++;
}

function _countAdjacentIce(uint16 x, uint16 y) internal view returns (uint8 count) {
    if (isIceDeposit(x,             uint16(y - 1))) count++;
    if (isIceDeposit(x,             uint16(y + 1))) count++;
    if (isIceDeposit(uint16(x - 1), y            )) count++;
    if (isIceDeposit(uint16(x + 1), y            )) count++;
}
```

### 1.5 Ice Adjacency Bonuses (Water, Oxygen, Habitation, Mining)

`adjacentIceTiles` (0–4) is stored in the Colony struct and baked in permanently at mint. It drives four distinct bonuses:

**1. Water capacity bonus** — free baseline water without extractors:

```solidity
uint32 iceWaterBonus = uint32(colonies[colonyId].adjacentIceTiles) * WATER_BONUS_PER_ICE_TILE;
// WATER_BONUS_PER_ICE_TILE = 200 water units
// Added to aggregates.waterCapacity at mint, before any extractors are built
```

**2. Oxygen production bonus** — ice electrolysis provides free O2:

```solidity
// Applied inside aggregates at mint:
uint32 iceOxygenBonus = uint32(colonies[colonyId].adjacentIceTiles) * O2_BONUS_PER_ICE_TILE;
// O2_BONUS_PER_ICE_TILE = 150 O2 units
// Added to aggregates.oxygenCapacity — supplements life support systems
```

**3. Habitation attractiveness bonus** — ice access means water security:

```solidity
// Applied inside residentialDemandMod():
uint256 iceHabBonus = uint256(colonies[colonyId].adjacentIceTiles) * 3; // up to +12 hMod points
hMod = _min(100, hMod + iceHabBonus);
```

Colonies near ice fields are more desirable to live in — settlers feel safer knowing water is abundant. This directly boosts habitation income.

**4. Mining logistics bonus** — ice provides cooling and hydrogen feedstock:

```solidity
// Applied inside miningDemandMod():
uint256 iceMiningBonus = uint256(colonies[colonyId].adjacentIceTiles) * 2; // up to +8 mMod points
mMod = _min(100, mMod + iceMiningBonus);
```

Ice-adjacent mining operations use water for cooling and hydrogen extraction. This partially offsets the dust penalty on habitation — an ice-adjacent mining colony is more productive than a dry one.

**Combined effect on colony strategy:**

| Adjacent ice tiles | Water | O2 | Hab bonus | Mining bonus | Character |
|---|---|---|---|---|---|
| 0 | +0 | +0 | +0 | +0 | Dry interior — must build all infrastructure |
| 1 | +200 | +150 | +3 hMod | +2 mMod | Ice-edge — mild advantages |
| 2 | +400 | +300 | +6 hMod | +4 mMod | Ice-field border — meaningful resource advantage |
| 3 | +600 | +450 | +9 hMod | +6 mMod | Deep ice — strong bonuses, only 1 trade neighbor |
| 4 | +800 | +600 | +12 hMod | +8 mMod | Ice island — best bonuses, zero trade neighbors |

Note: a tile surrounded by 4 ice tiles is isolated — best resource bonuses but no land neighbors, no trade deals possible. Maximum self-sufficiency at the cost of complete trade isolation.

### 1.6 Ice Deposits as a Resource Multiplier Between Colonies

Ice deposit tiles do not earn income and cannot be traded directly. Their effect is through the `adjacentIceTiles` count baked into each neighboring colony at mint — boosting water, oxygen, habitation demand, and mining demand permanently. Two colonies competing for a tile adjacent to an ice field are competing for a structural advantage across all four dimensions.

### 1.7 Centrality Bonus (Distance from Landing Site)

Colonies closer to the original landing site (geometric center) earn slightly more income. The landing site is where the first infrastructure was established — communication relays, supply depots, the original habitat. Proximity to this hub means better logistics and supply chain access.

```solidity
uint8  public constant MAX_CENTRALITY_BONUS = 10;  // +10% income at landing site
uint16 public constant CENTRALITY_RADIUS    = 64;  // bonus reaches 0% at 64 tiles out

function centralityModifier(uint256 colonyId) public view returns (uint256) {
    Colony storage c = colonies[colonyId];
    uint256 dx = c.surfaceX > SURFACE_CENTER ? c.surfaceX - SURFACE_CENTER : SURFACE_CENTER - c.surfaceX;
    uint256 dy = c.surfaceY > SURFACE_CENTER ? c.surfaceY - SURFACE_CENTER : SURFACE_CENTER - c.surfaceY;
    uint256 dist = dx > dy ? dx : dy; // Chebyshev distance

    if (dist >= CENTRALITY_RADIUS) return 100; // no bonus beyond radius
    return 100 + MAX_CENTRALITY_BONUS * (CENTRALITY_RADIUS - dist) / CENTRALITY_RADIUS;
}
```

Applied at claim as a final multiplier on gross income:

```solidity
// Inside claim():
uint256 centrality = centralityModifier(colonyId); // 100–110
uint256 gross = (c.accruedHabIncome * hMod / 100
              + c.accruedComIncome * cMod / 100
              + c.accruedMiningIncome * mMod / 100)
              * centrality / 100;
```

**Bonus gradient:**

| Distance from landing site | Centrality modifier | Character |
|---|---|---|
| 0 (landing site) | 110% | Hub — highest income potential |
| 16 tiles | 107% | Inner ring — strong logistics bonus |
| 32 tiles (launch edge) | 105% | Mid-range — moderate bonus |
| 48 tiles | 102% | Outer ring — minimal bonus |
| 64+ tiles | 100% | Frontier — no bonus, but less competition |

`CENTRALITY_RADIUS` is fixed at 64 — it does not scale with `surfaceRadius`. The bonus zone is permanently anchored to the original landing site. As the surface expands, new peripheral tiles have no centrality bonus but face less competition. The tradeoff: prime location vs. open space.

---

## 2. The Colony NFT (ERC-721)

Each colony is a unique NFT. Owning the NFT means owning the colony and all modules in it. Colonies can be bought and sold on any ERC-721 marketplace.

```solidity
struct Colony {
    uint16  surfaceX;
    uint16  surfaceY;
    uint8   adjacentIceTiles; // 0–4, computed at mint, permanent ice bonus
    uint32  foundedBlock;

    uint8   habTaxRate;     // 0–20%
    uint8   comTaxRate;
    uint8   miningTaxRate;

    // Running rate accumulator — split by zone category so demand modifiers
    // can be applied per-category at claim time (see §7 and §4)
    uint64  activeHabRatePerBlock;
    uint64  activeComRatePerBlock;
    uint64  activeMiningRatePerBlock;

    // Per-category accrued income (snapshotted on every write)
    uint128 accruedHabIncome;
    uint128 accruedComIncome;
    uint128 accruedMiningIncome;

    // Operating costs from services
    uint64  activeCostPerBlock;
    uint128 accruedCosts;

    uint32  lastWriteBlock;
    uint32  lastClaimBlock;
    uint128 treasury;
}

mapping(uint256 colonyId => Colony) public colonies;
```

Packs into 4 uint256 slots. Minting a colony: ~5 writes ≈ **$0.08 on Base**.

---

## 3. Zones (the Only On-Chain Primitive)

A "module" on-chain is just a zone placement. The player places a zone type at a coordinate — that's it. What renders visually (cramped bunk pod vs. luxury biodome, basic drill rig vs. automated smelter) is entirely client-side, derived from zone type, density tier, level, maturity, and neighboring context.

Civic structures (security stations, medical bays, research labs, etc.) are explicit zone types placed at specific coordinates. They follow the same placement mechanic as everything else.

Each zone placement has four fields:

- **x** — horizontal position inside the 128×128 colony grid (0–127)
- **y** — depth tier inside the colony grid (0 = surface, 127 = deepest — see §3.4)
- **zoneType** — what kind of module (habitation pod, mining rig, life support, etc.)
- **level** — upgrade tier (0 = base, up to 5)
- **birthBlock** — the block it was placed

Build state (under construction → operational → mature → upgrade eligible) is derived from age (`block.number - birthBlock`) and never stored. It changes automatically as blocks pass with no transaction required.

### 3.1 Packing

```solidity
// 8 bytes per zone:
// x:          uint8   (0–127, horizontal position)
// y:          uint8   (0–127, depth — 0 = surface, 127 = deepest)
// zoneType:   uint8   (up to 256 types, ~40 used)
// level:      uint8   (0–5 upgrade tier; 6 = demolished sentinel)
// birthBlock: uint32  (~272 years of headroom at Base 2s/block)
//
// 4 zones packed per uint256 slot (32 bytes / 8 bytes = 4)
// zone[i]: slot = i / 4, bit offset = (i % 4) * 64

mapping(uint256 colonyId => uint256[]) internal _zones;
mapping(uint256 colonyId => uint256)   public  zoneCount;
```

Placing a zone: 1 SSTORE ≈ **$0.03** (base cost, before excavation multiplier).
Upgrading a zone: 1 SLOAD + 1 SSTORE ≈ **$0.03**.

### 3.2 Build State (Derived, Never Stored)

```solidity
enum BuildState { UNDER_CONSTRUCTION, OPERATIONAL, MATURE, UPGRADE_ELIGIBLE }

// Blocks required to reach MATURE for each zone type — set at deploy
uint32[48] public MATURATION;

function getBuildState(uint8 zoneType, uint32 birthBlock)
    public view returns (BuildState)
{
    uint256 age    = block.number - birthBlock;
    uint256 mature = MATURATION[zoneType];

    if (age < mature / 2) return BuildState.UNDER_CONSTRUCTION; // 0–50%: silent
    if (age < mature)     return BuildState.OPERATIONAL;         // 50–100%: partial output
    if (age < mature * 3) return BuildState.MATURE;              // 100–300%: full output
    return BuildState.UPGRADE_ELIGIBLE;                          // 300%+: can level up
}
```

| State | Income Output |
|---|---|
| UNDER_CONSTRUCTION | 0% |
| OPERATIONAL | 50% of full rate |
| MATURE | 100% of full rate |
| UPGRADE_ELIGIBLE | 110% (longevity bonus) |

### 3.3 Levels

Level is an explicit stored upgrade — not automatic. An agent calls `upgradeZone(colonyId, index)`, pays $SIM, and the level increments. Requires UPGRADE_ELIGIBLE state.

After upgrading, `birthBlock` resets to `block.number` so the module re-matures at its new tier — extending productive life and increasing output rate.

```
Level 0: prefab
Level 1: reinforced
Level 2: expanded
Level 3: advanced
Level 4: hardened
Level 5: landmark (max)
```

### 3.4 Depth (Y-Axis)

The 128×128 colony grid uses the **x-axis for horizontal position** and the **y-axis for depth**. Y = 0 is the lunar surface. Y = 127 is the deepest excavation. The 8×8 sector grid (§10) has 8 rows, each representing a depth tier:

```
Tier 0 (y 0–15):    Surface   — solar access, high radiation, free excavation
Tier 1 (y 16–31):   Shallow   — moderate radiation, easy excavation
Tier 2 (y 32–47):   Upper     — low radiation, standard excavation cost
Tier 3 (y 48–63):   Mid       — minimal radiation, standard cost
Tier 4 (y 64–79):   Lower     — no radiation, moderate excavation cost
Tier 5 (y 80–95):   Deep      — no radiation, expensive excavation
Tier 6 (y 96–111):  Abyss     — no radiation, very expensive, geothermal access
Tier 7 (y 112–127): Core      — no radiation, maximum cost, geothermal bonus
```

**Excavation cost multiplier** — applied when placing a zone:

```solidity
uint16[8] public constant EXCAVATION_COST = [100, 100, 120, 140, 160, 200, 250, 300];

function placeZone(uint256 colonyId, uint8 x, uint8 y, uint8 zoneType) external {
    require(x < 128 && y < 128, "out of bounds");

    uint256 tier = uint256(y) / 16;
    uint256 cost = BASE_PLACE_COST * EXCAVATION_COST[tier] / 100;
    SimToken(simToken).burnFrom(msg.sender, cost);

    // Depth-restricted zone validation
    if (zoneType == SOLAR_ARRAY)     require(tier == 0, "solar requires surface");
    if (zoneType == GEOTHERMAL_TAP)  require(tier >= 6, "geothermal requires depth 6+");

    // ... pack and store zone
}
```

Surface is free to build on (no excavation). Deep tiers cost up to 3× base. This creates a real economic tradeoff: build shallow and deal with radiation, or pay more to build deep and safe.

**Radiation** is ambient by depth tier — not emitted by buildings. It suppresses habitation demand in shallow sectors. Radiation shielding modules (§5) reduce radiation in their surrounding sectors, making surface-level habitation viable at a cost. See §10 for how radiation interacts with the sector grid.

```solidity
// Base radiation per depth tier — decreases with depth
uint8[8] public constant BASE_RADIATION = [15, 10, 5, 2, 0, 0, 0, 0];
```

**Strategic implications of depth:**
- **Surface (tier 0):** Solar arrays work here and only here. Radiation is highest — habitation needs shielding. Spaceports must be at surface level.
- **Shallow (tier 1–2):** Cheaper to build than deep, some radiation. Good for mining and commercial with partial shielding.
- **Mid (tier 3–4):** The sweet spot for habitation. No/minimal radiation, moderate excavation cost. Most colonies will cluster habitation here.
- **Deep (tier 5–7):** Expensive but completely radiation-free. Geothermal power at tier 6–7. Ideal for critical infrastructure and high-density habitation if you can afford it.

---

## 4. Zone Economics and Interdependency

### What Each Zone Produces and Its Downside

**Habitation (replaces Residential)**
Produces population capacity and habitation tax income. Population is the fuel for everything else — no settlers means commercial has no customers and mining has no workers.

Downside: earns the least tax per zone on its own. Needs nearby jobs or nobody settles (low occupancy = low actual income). Dust from nearby mining drives settlers out. Requires services (security, medical, research) to maintain high occupancy, and services cost operating budget. Shallow habitation suffers radiation penalty unless shielded.

**Commercial**
Produces jobs and commercial tax income. Commercial tax rates can be set higher than habitation. Needs foot traffic — if your population is low relative to your commercial zone count, trading posts sit empty and earn a fraction of their rate.

Downside: entirely dependent on habitation population. No people = no customers = demand modifier near zero.

**Mining (replaces Industrial)**
Produces jobs and mining tax income — the highest tax rate of the three. Regolith processing and rare earth extraction pay well.

Downside: **dust**. Mining operations kick up regolith dust that damages equipment and degrades air quality. Dust suppresses habitation attractiveness. Settlers leave dusty colonies for cleaner ones. Fewer settlers means fewer workers, which reduces mining output further. Dense mining is the most profitable and the most damaging.

The dust downside is the central tension: mining income funds your colony but erodes your population base that commercial and habitation depend on. Research investment (labs, advanced research centers) gradually shifts mining to clean automated extraction — same income, far less dust.

**Services (security, emergency, medical, research, greenhouse)**
No direct income. They cost operating budget every block. Their value is indirect: high service coverage improves the habitation demand modifier, increasing hab occupancy, increasing hab and commercial tax income. A colony with no services will earn less from its hab zones even if it has plenty of jobs.

**Infrastructure (power, life support, water)**
No income. Required for everything else to work. Without mature life support, the colony depressurizes (all income halved). Without water, habitation and commercial income halved.

### Why You Can't Build Just One Type

| Colony Type | What Breaks |
|---|---|
| All habitation | No jobs → nobody settles → 0% occupancy → 0 tax income |
| All commercial | No customers → demand modifier = 0 → earns nothing |
| All mining | Dust drives everyone away → no workforce → mining earns nothing |
| All services | No income-producing zones → costs exceed revenue → bankruptcy |

The system is circular. Habitation needs jobs from commercial and mining. Commercial needs customers from habitation. Mining needs workers from habitation. Dust from mining drives settlers away, which starves commercial and mining of customers and workers.

**The Waste Heat City**
A valid specialization: thermal recyclers, waste processing, light mining. High dust, near-zero hab attractiveness. Income comes primarily from mining taxes and neighbor waste disposal deals ($SIM per block from neighbor colonies that export their waste). Capped by the fact that low population = low mining demand modifier, so mining income has a ceiling without some hab base.

### Demand Modifiers

Income earned by each zone category is multiplied by a demand modifier at claim time. Modifiers are 0–100 (percentage of full rate). Stored aggregate counters make this O(1) math — no zone iteration at claim.

**Habitation demand modifier:**
```
jobCoverage     = min(100, (comJobs + miningJobs) × 100 / habCapacity)
dustPenalty     = min(60, effectiveDust / 10)         ← reduced by research score
radiationPenalty = from spatial sector sweep (§10)     ← reduced by shielding
serviceBonus    = min(20, serviceScore / 5)
greenBonus      = min(10, greenScore / 5)             ← from greenhouses/arboreta
iceBonus        = adjacentIceTiles × 3                ← up to +12

hMod = clamp(0–100, jobCoverage − dustPenalty − radiationPenalty + serviceBonus + greenBonus + iceBonus)
```

**Commercial demand modifier:**
```
cMod = min(100, population × 100 / commercialCapacity)
```
Population must be proportional to commercial supply or trading posts sit empty.

**Mining demand modifier:**
```
workerRatio = min(100, population × 100 / miningCapacity)
eduBonus    = min(20, researchScore / 5)   ← advanced research enables automated, more profitable mining
iceBonus    = adjacentIceTiles × 2         ← up to +8

mMod = min(100, workerRatio + eduBonus + iceBonus)
```

### Zone Density and Dust

Each of the three income zone types has three density tiers. Higher density = more income and (for mining) more dust:

| Zone | Light | Medium | Dense |
|---|---|---|---|
| Habitation | 100 pop cap, low tax | 300 pop cap, med tax | 700 pop cap, high tax |
| Commercial | 50 jobs, low tax | 150 jobs, med tax | 400 jobs, high tax |
| Mining | 80 jobs, low tax, **low dust** | 200 jobs, med tax, **med dust** | 500 jobs, high tax, **high dust** |

Effective mining dust = `rawDust × (100 − researchScore) / 100`. At research score 80, dense mining produces only 20% of base dust (automated clean extraction).

### Colony Aggregate Counters (O(1) Demand)

To keep demand calculation O(1) at claim, counters are maintained per colony and updated on every `activateZone` and `demolish` call:

```solidity
struct ColonyAggregates {
    // Population and zones
    uint32 matureHabCapacity;     // total population slots from mature habitation
    uint32 matureComJobs;         // jobs from mature commercial
    uint32 matureComCapacity;     // commercial slot count
    uint32 matureMiningJobs;      // jobs from mature mining
    uint32 matureMiningCapacity;  // mining slot count
    uint32 matureMiningDust;      // raw dust units (reduced by researchScore at use)

    // Research, services, green
    uint32 researchScore;         // 0–100, from research labs / advanced research
    uint32 serviceScore;          // 0–100, from security/emergency/medical coverage
    uint32 greenScore;            // 0–100, from greenhouses and arboreta

    // Crime
    uint32 baseCrime;             // raw crime generated (mining density + dense zones)
    uint32 securityCapacity;      // crime reduction units from mature security stations
    bool   hasBrig;               // brig present (required above BRIG_POP_THRESHOLD)

    // Power (MW)
    uint32 powerCapacity;         // MW produced by mature power sources (local)
    uint32 powerDemand;           // MW consumed by all mature modules

    // Oxygen (O2 units)
    uint32 oxygenCapacity;        // O2 produced by mature life support + ice bonus
    uint32 oxygenDemand;          // O2 consumed by all occupied hab/com/mining zones

    // Water
    uint32 waterCapacity;         // units produced by mature ice extractors + ice bonus
    uint32 waterDemand;           // units consumed by mature hab + commercial

    // Waste (units per block)
    uint32 wasteGenPerBlock;      // generated by population (scales with matureHabCapacity)
    uint32 wasteCapPerBlock;      // handled by waste storage/thermal recyclers/waste-to-energy
}

mapping(uint256 colonyId => ColonyAggregates) public aggregates;
```

Updated on every `activateZone` (increment) and `demolish` (decrement). All demand modifiers and resource balance checks are pure arithmetic from these counters — no loops.

---

## 5. Zone Types

Tunnels, power conduits, and pipes have **no on-chain effect** — power and life support are aggregate capacity math, not network connectivity. They are cosmetic: the indexer renders them in the UI but the contract ignores them. Only the zone types below carry on-chain mechanics.

| ID | Zone | MW / Water / O2 / Crime | Notes |
|---|---|---|---|
| 0 | (empty / demolished) | — | — |
| 1 | Tunnel *(cosmetic)* | — | UI only — no on-chain effect |
| 2 | Power Conduit *(cosmetic)* | — | UI only — power is aggregate capacity |
| 3 | Ice Extractor | +600 water | Must be on ice-adjacent colony |
| 4 | Water Recycler | +300 water | Anywhere, lower capacity than extractor |
| 5 | Pipe *(cosmetic)* | — | UI only — water is aggregate capacity |
| **Habitation** | | | |
| 6 | Habitation Pod | −50 water, −20 O2 | 100 pop cap |
| 7 | Habitation Module | −150 water, −60 O2 | 300 pop cap |
| 8 | Biodome | −350 water, −140 O2 | 700 pop cap |
| **Commercial** | | | |
| 9 | Trade Post | −30 water, −10 O2 | 50 jobs |
| 10 | Market Hall | −80 water, −30 O2 | 150 jobs |
| 11 | Commerce Dome | −200 water, −80 O2 | 400 jobs |
| **Mining** | | | |
| 12 | Mining Rig (Light) | +20 crime, −30 MW, −15 O2 | 80 jobs, low dust |
| 13 | Processing Plant | +50 crime, −80 MW, −40 O2 | 200 jobs, med dust |
| 14 | Smelter Complex | +100 crime, −200 MW, −100 O2 | 500 jobs, high dust |
| **Waste** | | | |
| 15 | Waste Storage | — | +500 waste cap/block |
| 16 | Thermal Recycler | −50 MW, +dust | +1,000 waste cap/block |
| 17 | Recycling Module | −30 MW | Reduces waste gen 25% |
| **Green / Morale** | | | |
| 18 | Greenhouse (Small) | +30 O2 | +greenScore, small O2 production |
| 19 | Arboretum | +100 O2 | +greenScore (3×), meaningful O2 production |
| **Services** | | | |
| 20 | Security Station | −30 MW | Covers 3×3 sectors; +securityCapacity |
| 21 | Emergency Station | −20 MW | Decompression/fire response; +serviceScore |
| 22 | Medical Bay | −40 MW | +health, +serviceScore, handles radiation sickness |
| 23 | Research Lab | −30 MW | +researchScore |
| 24 | Advanced Research Center | −50 MW | +researchScore (2×), unlocks high-tech |
| **Transit** | | | |
| 25 | Rail Station | −60 MW | Covers 3×3 sectors; +transitSectors |
| 26 | Maglev Station | −80 MW | Covers 3×3 sectors; +transitSectors (weight 2) |
| 27 | Cargo Terminal | −100 MW | Covers 5×5 sectors; +transitSectors + mining demand bonus |
| 28 | Spaceport | −150 MW | Covers 5×5 sectors; +transitSectors + commercial bonus. **Surface only (tier 0)** |
| 29 | Brig | −60 MW | Required above BRIG_POP_THRESHOLD |
| **Waste-to-Energy** | | | |
| 30 | Waste-to-Energy Plant | **+400 MW**, +dust | +2,000 waste cap/block |
| **Life Support** | | | |
| 31 | Life Support (Basic) | **+500 O2**, −50 MW | No unlock required |
| 32 | Life Support (Advanced) | **+1,200 O2**, −100 MW | researchScore > 30 |
| **Power** | | | |
| 33 | Solar Array | **+300 MW**, no dust | **Surface only (tier 0)** |
| 34 | RTG Bank | **+400 MW**, no dust | No unlock; low output, any depth |
| 35 | Nuclear Reactor | **+5,000 MW**, no dust | researchScore > 40; meltdown risk |
| 36 | Geothermal Tap | **+1,500 MW**, no dust | **Tier 6–7 only**; no unlock |
| 37 | Microwave Receiver | **+4,000 MW**, no dust | researchScore > 70; **surface only** |
| 38 | Fusion Reactor | **+10,000 MW**, no dust | researchScore > 90 |
| **Radiation Shielding** | | | |
| 39 | Radiation Shield | −40 MW | Covers 3×3 sectors; reduces radiation in shallow tiers |

---

## 5b. Maturation Periods (Base, 2s blocks)

| Zone | Blocks to MATURE | Real Time |
|---|---|---|
| Tunnel, Power Conduit, Pipe *(cosmetic — no activate call needed)* | — | — |
| Life Support, Ice Extractor | 21,600 | 12 hours |
| Greenhouses | 21,600 | 12 hours |
| Habitation Pod, Trade Post | 43,200 | 1 day |
| Services (security/emergency/medical/research) | 43,200 | 1 day |
| Mining (all densities) | 86,400 | 2 days |
| Habitation Module / Market Hall | 129,600 | 3 days |
| Biodome / Commerce Dome | 259,200 | 6 days |
| Spaceport / Cargo Terminal | 259,200 | 6 days |
| Radiation Shield | 43,200 | 1 day |

---

## 6. Colony-Level Derived Stats (View Functions, No Storage)

All game metrics — population, dust, research, power status, oxygen status — are pure view functions. They read the zone array and `block.number` and return results. No storage written, no gas cost to read.

```solidity
function colonyStats(uint256 colonyId) public view returns (
    uint256 population,
    uint256 dustLevel,
    uint256 researchScore,
    bool    hasPower,
    bool    hasOxygen,
    bool    hasWater,
    uint256 wastePerBlock,
    uint256 wasteCapacity
);

function attractiveness(uint256 colonyId) public view returns (uint256);
function population(uint256 colonyId)     public view returns (uint256);
```

These iterate the zone array once. At 500 zones: ~125 warm SLOADs ≈ free as view calls, ~262k gas if called inside a transaction.

---

## 7. Income: Running Rate Accumulator

The claim function is **O(1) regardless of colony size**. It never iterates zones.

### How It Works

The colony stores per-category rate and accrued income fields that together always answer "how much has this colony earned, broken down by zone type":

- `activeHabRatePerBlock` / `activeComRatePerBlock` / `activeMiningRatePerBlock` — $SIM/block flowing from mature zones in each category
- `accruedHabIncome` / `accruedComIncome` / `accruedMiningIncome` — income banked per category as of `lastWriteBlock`
- `lastWriteBlock` — block when colony state last changed

Splitting by category lets demand modifiers (§4) be applied per-category at claim time — habitation income is multiplied by `hMod`, commercial by `cMod`, mining by `mMod`.

Whenever anything changes, a snapshot banks the current rate into accrued income first:

```solidity
function _snapshot(uint256 colonyId) internal {
    Colony storage c = colonies[colonyId];
    uint256 elapsed = block.number - c.lastWriteBlock;

    c.accruedHabIncome    += uint128(uint256(c.activeHabRatePerBlock)    * elapsed);
    c.accruedComIncome    += uint128(uint256(c.activeComRatePerBlock)    * elapsed);
    c.accruedMiningIncome += uint128(uint256(c.activeMiningRatePerBlock) * elapsed);
    c.accruedCosts        += uint128(uint256(c.activeCostPerBlock)       * elapsed);

    c.lastWriteBlock = uint32(block.number);
}
```

Called automatically at the start of every write — place, upgrade, demolish, activate, tax change, ordinance toggle.

### Activating a Matured Zone

When a zone reaches OPERATIONAL or MATURE state it does not automatically earn income. An agent must call `activateZone(colonyId, zoneIndex)` to register it:

```solidity
function activateZone(uint256 colonyId, uint256 zoneIndex) external {
    (, , uint8 zoneType, uint8 level, uint32 birthBlock) =
        _decodeZone(colonyId, zoneIndex);

    require(getBuildState(zoneType, birthBlock) >= BuildState.OPERATIONAL,
            "not yet operational");

    _snapshot(colonyId); // bank current income before changing rates

    uint64 rate = _zoneRate(zoneType, level, colonies[colonyId]);
    Colony storage c = colonies[colonyId];

    // Route to the correct income bucket
    if      (isHabitation(zoneType)) c.activeHabRatePerBlock    += rate;
    else if (isCommercial(zoneType)) c.activeComRatePerBlock    += rate;
    else if (isMining(zoneType))     c.activeMiningRatePerBlock += rate;
    else if (isService(zoneType))    c.activeCostPerBlock       += COST[zoneType][level];

    // Update aggregate counters used by demand modifiers
    ColonyAggregates storage a = aggregates[colonyId];
    if (isHabitation(zoneType)) a.matureHabCapacity     += HAB_CAP[zoneType][level];
    if (isCommercial(zoneType)) { a.matureComJobs       += COM_JOBS[zoneType][level];
                                  a.matureComCapacity   += COM_CAP[zoneType][level]; }
    if (isMining(zoneType))     { a.matureMiningJobs    += MINING_JOBS[zoneType][level];
                                  a.matureMiningCapacity += MINING_CAP[zoneType][level];
                                  a.matureMiningDust     += DUST[zoneType][level]; }
    if (isResearch(zoneType))   a.researchScore = _recalcResearch(colonyId);
    if (isService(zoneType))    a.serviceScore  = _recalcServiceScore(colonyId);
    if (isGreen(zoneType))      a.greenScore    = _recalcGreenScore(colonyId);

    // Oxygen and power aggregates
    a.oxygenCapacity += O2_PRODUCTION[zoneType][level];
    a.oxygenDemand   += O2_DEMAND[zoneType][level];
    a.powerCapacity  += POWER_PRODUCTION[zoneType][level];
    a.powerDemand    += POWER_DEMAND[zoneType][level];

    emit ZoneActivated(colonyId, zoneIndex, zoneType, rate);
}
```

Agents are incentivized to call this promptly — every block of delay is income lost. A small $SIM bounty enables a keeper market where anyone can activate any colony's zones.

When a zone is demolished or upgraded, `_snapshot` runs first and the old rate and aggregate values are decremented before the new state takes effect.

### The Claim

```solidity
function claim(uint256 colonyId) external {
    require(ownerOf(colonyId) == msg.sender, "not owner");

    _snapshot(colonyId); // bank income at current rates up to this block

    Colony storage c = colonies[colonyId];

    // Demand modifiers: O(1) math from stored ColonyAggregates (no zone iteration)
    uint256 hMod = habDemandMod(colonyId);     // 0–100
    uint256 cMod = commercialDemandMod(colonyId);
    uint256 mMod = miningDemandMod(colonyId);

    // Apply demand to each income bucket, then centrality bonus
    uint256 centrality = centralityModifier(colonyId); // 100–110
    uint256 gross = (c.accruedHabIncome * hMod / 100
                  + c.accruedComIncome * cMod / 100
                  + c.accruedMiningIncome * mMod / 100)
                  * centrality / 100;

    // Resource penalties
    uint256 o2Mod    = hasOxygen(colonyId) ? 100 : 50; // depressurization
    uint256 brownout = hasPower(colonyId)  ? 100 : 50; // brownout
    uint256 drought  = hasWater(colonyId)  ? 100 : 50; // water crisis
    gross = gross * o2Mod / 100 * brownout / 100 * drought / 100;

    uint256 deals = _dealIncome(colonyId); // neighbor trade deal earnings

    int256 net = int256(gross) + int256(deals) - int256(c.accruedCosts);

    // Clear buckets
    c.accruedHabIncome    = 0;
    c.accruedComIncome    = 0;
    c.accruedMiningIncome = 0;
    c.accruedCosts        = 0;
    c.lastClaimBlock      = uint32(block.number);

    if (net > 0) {
        uint256 n = uint256(net);
        c.treasury += uint128(n / 10);               // 10% to colony treasury
        SimToken(simToken).mint(msg.sender, n - n / 10);
    }

    emit Claimed(colonyId, net, block.number);
}
```

**Claim gas: ~185k regardless of colony size. ≈ $0.28 on Base.** See §10 for the full breakdown.

Demand modifiers are pure arithmetic over `ColonyAggregates` (including ice bonuses from `adjacentIceTiles`) — no zone iteration. Centrality modifier is pure math from stored colony coordinates. Spatial modifiers read 8 fixed storage slots and sweep 64 sectors in one arithmetic pass. The only variable-count operation is `_dealIncome` (0–8 active deals per colony). Zone count does not affect claim cost.

### Operating Costs

Service modules (security, emergency, medical, research) have a per-block operating cost deducted at claim. Stored as a separate running rate:

```solidity
uint64 activeCostPerBlock; // mirrors activeRatePerBlock but for costs
```

Updated the same way — snapshot then adjust — whenever a service module is placed, activated, upgraded, or demolished.

---

## 8. Population and Colony Competition

Population is a pure view. No writes. No global sync.

### Attractiveness Score

```solidity
function attractiveness(uint256 colonyId) public view returns (uint256) {
    if (!_hasOxygen(colonyId) || !_hasPower(colonyId) || !_hasWater(colonyId)) return 0;

    uint256 score = 0;
    uint256 dust  = 0;
    uint256 res   = _researchScore(colonyId);
    uint256 count = zoneCount[colonyId];

    for (uint256 i = 0; i < count; i++) {
        (, , uint8 zoneType, uint8 level, uint32 birthBlock) =
            _decodeZone(colonyId, i);
        if (getBuildState(zoneType, birthBlock) < BuildState.OPERATIONAL) continue;

        if (isHabitation(zoneType)) score += HAB_SCORE[zoneType]     * (level + 1);
        if (isService(zoneType))    score += SERVICE_SCORE[zoneType] * (level + 1);
        if (isGreen(zoneType))      score += GREEN_SCORE             * (level + 1);
        if (isMining(zoneType))     dust  += DUST[zoneType];
    }

    // High research reduces mining dust (automated clean extraction)
    dust = dust * (100 - res) / 100;

    return score > dust ? score - dust : 0;
}
```

### Population

```solidity
uint256 public constant BASELINE_ATTRACTIVENESS = 1000;

function population(uint256 colonyId) public view returns (uint256) {
    uint256 cap   = _habCapacity(colonyId);
    uint256 score = attractiveness(colonyId);
    if (cap == 0 || score == 0) return 0;

    // At baseline → 50% occupancy. Above baseline → higher. Capped at 100%.
    uint256 occupancy = score * 5e17 / BASELINE_ATTRACTIVENESS;
    if (occupancy > 1e18) occupancy = 1e18;

    return cap * occupancy / 1e18;
}
```

As blocks pass, build states transition automatically. A mining rig maturing this block raises dust, dropping attractiveness, dropping population — all reflected instantly in the view with no transaction.

---

## 9. Crime

### How Crime Works

Crime is generated by mining operations and dense development. It is reduced by security stations. Without adequate security coverage, crime suppresses commercial income (people avoid trading in dangerous areas) and habitation income (settlers leave unsafe colonies). Above a population threshold, a Brig is required or security effectiveness is halved — criminals are released back into the colony.

### What Generates Crime

```
baseCrime = (matureMiningLight  × 20)
          + (matureMiningMed    × 50)
          + (matureMiningDense  × 100)
          + (matureHabDense     × 10)   ← overcrowding
          + (matureComDense     × 15)   ← opportunity crime in busy commercial
```

Updated in `ColonyAggregates.baseCrime` on every `activateZone` / `demolish`.

### What Reduces Crime

Each mature security station contributes a fixed `securityCapacity` (e.g., 80 crime units). Updated in `ColonyAggregates.securityCapacity`.

```solidity
function crimeScore(uint256 colonyId) public view returns (uint256) {
    ColonyAggregates memory a = aggregates[colonyId];

    uint256 raw = a.baseCrime > a.securityCapacity
                ? a.baseCrime - a.securityCapacity
                : 0;

    // Brig requirement: without brig above threshold, crime 50% worse
    bool needsBrig = (a.matureHabCapacity > BRIG_POP_THRESHOLD);
    if (needsBrig && !a.hasBrig) raw = raw * 3 / 2;

    return raw;
}
```

Security funding level (set via tax/budget config) scales `securityCapacity`:
- 50% funding → securityCapacity × 0.5
- 100% funding → securityCapacity × 1.0
- 150% funding → securityCapacity × 1.3

### Crime's Effect on Demand Modifiers

Crime is subtracted from both habitation and commercial demand modifiers:

```
hMod penalty: min(30, crimeScore / 3)   ← up to 30 point drag on habitation
cMod penalty: min(20, crimeScore / 5)   ← up to 20 point drag on commercial
```

These are already applied inside `habDemandMod()` and `commercialDemandMod()`. A high-crime colony earns less from both hab and commercial zones, incentivizing security investment.

### Ordinances That Affect Crime

| Ordinance | Crime Effect |
|---|---|
| Sector Watch | −10 crimeScore |
| Curfew Protocol | −5 crimeScore |
| Recreation Program | −8 crimeScore, +researchScore |
| Gambling License | +15 crimeScore, +revenue |

---

## 10. Spatial Mechanics (Sector Grid)

### Why Sectors?

Crime, dust, and radiation are proximity-based: a security station protects nearby sectors but not distant ones; a smelter dusts its neighbors; surface sectors are irradiated. True per-cell distance math would require O(n²) zone iteration at claim — untenable. Instead the 128×128 colony grid is divided into an **8×8 = 64 sector grid**. Coverage is tracked at sector granularity, not cell granularity — cheap to update on placement, cheap to sweep at claim. All 4-bit overlays fit in a single uint256 slot each.

### Grid Layout

```
Colony grid:   128 × 128 cells  (uint8 x = horizontal, uint8 y = depth, 0–127)
Sector grid:   8 × 8 sectors    (each sector = 16 × 16 cells)
Sector index:  sectorIdx = (y / 16) * 8 + (x / 16)   // 0–63
Sector row:    depthTier = y / 16                      // 0–7 (surface to core)
```

Coverage radius: a security station or mining zone covers a **3×3 sector square** centered on its own sector — 9 sectors, approximately a 48×48 cell area (~14% of the colony per station).

### Storage

```solidity
// 4 bits per sector × 64 sectors = 256 bits → 1 × uint256 slot
// Sector s: bit offset = s * 4
mapping(uint256 colonyId => uint256) public securitySectors;   // security coverage count, 0–15
mapping(uint256 colonyId => uint256) public dustSectors;       // mining dust level, 0–15
mapping(uint256 colonyId => uint256) public transitSectors;    // transit coverage level, 0–15
mapping(uint256 colonyId => uint256) public shieldSectors;     // radiation shielding, 0–15

// 8 bits per sector × 64 sectors = 512 bits → 2 × uint256 slots
// Sector s: slot = s/32, bit offset = (s%32)*8
mapping(uint256 colonyId => uint256[2]) public habSectors;     // habitation zone count, 0–255
mapping(uint256 colonyId => uint256[2]) public comSectors;     // commercial zone count, 0–255
```

Total storage per city: 8 uint256 slots — all read in a single pass at claim.

### Updating on activateZone

One additional code path runs in `activateZone` for security, mining, transit, shielding, and zone placements:

```solidity
uint256 sx = uint8(x) / 16;
uint256 sy = uint8(y) / 16;  // also = depth tier

if (zoneType == SECURITY_STATION) {
    for (uint256 ry = (sy > 0 ? sy-1 : 0); ry <= _min(sy+1, 7); ry++) {
        for (uint256 rx = (sx > 0 ? sx-1 : 0); rx <= _min(sx+1, 7); rx++) {
            uint256 s      = ry * 8 + rx;
            uint256 offset = s * 4;
            uint256 cur    = (securitySectors[colonyId] >> offset) & 0xF;
            if (cur < 15) securitySectors[colonyId] =
                (securitySectors[colonyId] & ~(uint256(0xF) << offset)) | ((cur+1) << offset);
        }
    }
}

if (isMining(zoneType)) {
    uint256 dustAdd = (zoneType == MINING_LIGHT) ? 1 : (zoneType == MINING_MED) ? 2 : 4;
    for (uint256 ry = (sy > 0 ? sy-1 : 0); ry <= _min(sy+1, 7); ry++) {
        for (uint256 rx = (sx > 0 ? sx-1 : 0); rx <= _min(sx+1, 7); rx++) {
            uint256 s      = ry * 8 + rx;
            uint256 offset = s * 4;
            uint256 cur    = (dustSectors[colonyId] >> offset) & 0xF;
            uint256 next   = cur + dustAdd > 15 ? 15 : cur + dustAdd;
            if (next != cur) dustSectors[colonyId] =
                (dustSectors[colonyId] & ~(uint256(0xF) << offset)) | (next << offset);
        }
    }
}

if (zoneType == RADIATION_SHIELD) {
    for (uint256 ry = (sy > 0 ? sy-1 : 0); ry <= _min(sy+1, 7); ry++) {
        for (uint256 rx = (sx > 0 ? sx-1 : 0); rx <= _min(sx+1, 7); rx++) {
            uint256 s      = ry * 8 + rx;
            uint256 offset = s * 4;
            uint256 cur    = (shieldSectors[colonyId] >> offset) & 0xF;
            if (cur < 15) shieldSectors[colonyId] =
                (shieldSectors[colonyId] & ~(uint256(0xF) << offset)) | ((cur+1) << offset);
        }
    }
}

// Transit hubs: radius depends on type
// Rail/Maglev → 3×3 sectors; Cargo Terminal/Spaceport → 5×5 sectors
if (zoneType == RAIL_STATION || zoneType == MAGLEV_STATION
    || zoneType == CARGO_TERMINAL || zoneType == SPACEPORT) {

    uint256 radius = (zoneType == CARGO_TERMINAL || zoneType == SPACEPORT) ? 2 : 1;
    uint256 weight = (zoneType == MAGLEV_STATION) ? 2 : 1;

    uint256 ryMin = sy >= radius ? sy - radius : 0;
    uint256 ryMax = _min(sy + radius, 7);
    uint256 rxMin = sx >= radius ? sx - radius : 0;
    uint256 rxMax = _min(sx + radius, 7);

    for (uint256 ry = ryMin; ry <= ryMax; ry++) {
        for (uint256 rx = rxMin; rx <= rxMax; rx++) {
            uint256 s      = ry * 8 + rx;
            uint256 offset = s * 4;
            uint256 cur    = (transitSectors[colonyId] >> offset) & 0xF;
            uint256 next   = cur + weight > 15 ? 15 : cur + weight;
            if (next != cur) transitSectors[colonyId] =
                (transitSectors[colonyId] & ~(uint256(0xF) << offset)) | (next << offset);
        }
    }
}

if (isHabitation(zoneType)) {
    uint256 s      = sy * 8 + sx;
    uint256 slot   = s / 32;
    uint256 offset = (s % 32) * 8;
    uint256 cur    = (habSectors[colonyId][slot] >> offset) & 0xFF;
    if (cur < 255) habSectors[colonyId][slot] =
        (habSectors[colonyId][slot] & ~(uint256(0xFF) << offset)) | ((cur+1) << offset);
}

if (isCommercial(zoneType)) {
    uint256 s      = sy * 8 + sx;
    uint256 slot   = s / 32;
    uint256 offset = (s % 32) * 8;
    uint256 cur    = (comSectors[colonyId][slot] >> offset) & 0xFF;
    if (cur < 255) comSectors[colonyId][slot] =
        (comSectors[colonyId][slot] & ~(uint256(0xFF) << offset)) | ((cur+1) << offset);
}
```

Demolish mirrors activate: same loops, decrement instead of increment.

**Gas cost for sector updates** (in addition to base activateZone cost):

With 8×8 sectors, all 4-bit overlays (security, dust, transit, shielding) fit in a single uint256 — every update is a single SLOAD+SSTORE regardless of coverage radius.

| Zone type | Sectors touched | Slots touched | Additional gas |
|---|---|---|---|
| Security station | 9 (3×3) | 1 slot | +5k warm |
| Mining | 9 (3×3) | 1 slot | +5k warm |
| Radiation Shield | 9 (3×3) | 1 slot | +5k warm |
| Rail / Maglev Station | 9 (3×3) | 1 slot | +5k warm |
| Cargo Terminal / Spaceport | 25 (5×5) | 1 slot | +5k warm |
| Habitation | 1 | 1 slot of habSectors | +3k warm |
| Commercial | 1 | 1 slot of comSectors | +3k warm |

### Spatial Demand Modifiers at Claim

After loading all sector slots, the claim function sweeps all 64 sectors in one arithmetic pass (no storage reads inside the loop). **Radiation is computed per-sector from the depth tier constant + shielding overlay — no extra storage.**

```solidity
function _spatialModifiers(uint256 colonyId)
    internal view
    returns (
        uint256 securedPct,     // % of hab zones in security-covered sectors
        uint256 dustyPct,       // % of hab zones in dusty sectors
        uint256 radiatedPct,    // % of hab zones in irradiated (unshielded) sectors
        uint256 transitHabPct,  // % of hab zones in transit-covered sectors
        uint256 transitComPct   // % of commercial zones in transit-covered sectors
    )
{
    uint256[2] memory hs  = habSectors[colonyId];        // 2 SLOADs
    uint256[2] memory cs  = comSectors[colonyId];        // 2 SLOADs
    uint256 sec = securitySectors[colonyId];              // 1 SLOAD
    uint256 dst = dustSectors[colonyId];                  // 1 SLOAD
    uint256 sh  = shieldSectors[colonyId];                // 1 SLOAD
    uint256 tr  = transitSectors[colonyId];               // 1 SLOAD

    uint256 totalHab = 0; uint256 totalCom = 0;
    uint256 securedHab = 0; uint256 dustyHab = 0;
    uint256 radiatedHab = 0;
    uint256 transitHab = 0; uint256 transitCom = 0;

    for (uint256 s = 0; s < 64; s++) {
        uint256 habCount = (hs [s/32] >> ((s%32)*8)) & 0xFF;
        uint256 comCount = (cs [s/32] >> ((s%32)*8)) & 0xFF;
        uint256 secured  = (sec >> (s*4)) & 0xF;
        uint256 dusty    = (dst >> (s*4)) & 0xF;
        uint256 shielded = (sh  >> (s*4)) & 0xF;
        uint256 transit  = (tr  >> (s*4)) & 0xF;

        // Radiation: ambient by depth tier, reduced by shielding
        uint256 depthTier   = s / 8;  // sector row = depth tier
        uint256 baseRad     = BASE_RADIATION[depthTier];
        uint256 effectiveRad = baseRad > shielded ? baseRad - shielded : 0;

        totalHab += habCount; totalCom += comCount;
        if (secured > 0)      securedHab  += habCount;
        if (dusty > 0)        dustyHab    += habCount;
        if (effectiveRad > 0) radiatedHab += habCount;
        if (transit > 0)      { transitHab += habCount; transitCom += comCount; }
    }

    securedPct   = totalHab > 0 ? securedHab  * 100 / totalHab : 100;
    dustyPct     = totalHab > 0 ? dustyHab    * 100 / totalHab : 0;
    radiatedPct  = totalHab > 0 ? radiatedHab * 100 / totalHab : 0;
    transitHabPct = totalHab > 0 ? transitHab * 100 / totalHab : 0;
    transitComPct = totalCom > 0 ? transitCom * 100 / totalCom : 0;
}
```

**Applied inside `claim()`** after the base demand modifiers:

```solidity
(uint256 securedPct, uint256 dustyPct, uint256 radiatedPct,
 uint256 transitHabPct, uint256 transitComPct) = _spatialModifiers(colonyId);

// Unsecured fraction amplifies the crime penalty
uint256 rawCrime      = crimeScore(colonyId);
uint256 spatialCrime  = rawCrime * (100 - securedPct) / 100;
hMod = hMod > spatialCrime / 3 ? hMod - spatialCrime / 3 : 0;
cMod = cMod > spatialCrime / 5 ? cMod - spatialCrime / 5 : 0;

// Dusty fraction suppresses habitation demand (up to −50 pts at 100% dusty)
uint256 dustPenalty = dustyPct / 2;
hMod = hMod > dustPenalty ? hMod - dustPenalty : 0;

// Radiation penalty — hab zones in irradiated sectors (up to −40 pts)
uint256 radPenalty = radiatedPct * 40 / 100;
hMod = hMod > radPenalty ? hMod - radPenalty : 0;

// Transit coverage boosts commercial and habitation demand (up to +15 pts)
cMod = _min(100, cMod + transitComPct * 15 / 100);
hMod = _min(100, hMod + transitHabPct * 10 / 100);
```

**Strategic implications (8×8 sector grid + depth):**
- **Security placement**: one station covers 9/64 sectors (~14%). Two well-placed stations cover ~40% of the colony. Coverage gaps directly cost income.
- **Mining zoning**: cluster mining rigs at one depth level and one side — habitation on the opposite side and a different depth earns more. Mixing mining and habitation kills hab income.
- **Radiation shielding**: surface habitation (tier 0, base radiation 15) needs multiple shield modules to be viable. Alternatively, build hab at tier 3+ where radiation is minimal. Shields make surface hab possible but expensive — a strategic choice.
- **Depth specialization**: surface = solar power + spaceport. Mid = habitation. Deep = mining + geothermal. This vertical zoning emerges naturally from the depth tier constraints and radiation gradient.
- **Transit hubs**: a cargo terminal covers 5×5 = 25/64 sectors (~39%). One spaceport (surface only) can cover nearly half the colony vertically, boosting commercial income across multiple depth tiers.

### Claim Gas Breakdown (Full, With Spatial + Centrality + Depth)

| Step | Operations | Gas (cold) |
|---|---|---|
| `_snapshot` | 5 SLOADs + 5 SSTOREs | ~73k |
| Demand modifiers (hMod/cMod/mMod) | 1 SLOAD (ColonyAggregates) + ice bonuses | ~8k |
| Centrality modifier | 1 SLOAD (colony coords), pure math | ~3k |
| Spatial + radiation (8 SLOADs + 64-iter bit loop) | 8 SLOADs, no in-loop SLOADs | ~24k |
| Resource penalties (O2/power/water) | 3 view calls from aggregates | ~4k |
| Trade deal income (avg 4 deals) | ~8 SLOADs | ~28k |
| Clear buckets + ERC-20 mint | 5 SSTOREs + external call | ~47k |
| **Total** | | **~185k ≈ $0.28 on Base** |

The 64-sector sweep with radiation lookup contributes only ~5k gas (pure arithmetic from constants + loaded memory). The bottleneck remains `_snapshot` SSTOREs and the ERC-20 mint.

**O(1) in zone count**: a colony with 10 zones costs the same to claim as one with 500. The ~185k gas ceiling is fixed by the 8-slot sector read plus standard EVM overhead.

---

## 11. Life Support, Power, and Water

### Life Support (Oxygen) — The Primary Critical Resource

Every occupied module consumes oxygen. Oxygen is produced by life support systems and supplemented by ice adjacency bonuses. If total O2 capacity falls below total demand, the colony enters **depressurization**: all income rates are halved until surplus is restored.

This is the most critical resource — without breathable air, nothing works. Life support is the first thing you build.

```solidity
function oxygenSurplus(uint256 colonyId) public view returns (int256) {
    ColonyAggregates memory a = aggregates[colonyId];
    uint256 imported = _importedOxygen(colonyId); // from active trade deals
    return int256(a.oxygenCapacity + imported) - int256(a.oxygenDemand);
}

function hasOxygen(uint256 colonyId) public view returns (bool) {
    return oxygenSurplus(colonyId) >= 0;
}
```

### Power — Secondary Critical Resource

Every mature module consumes power (MW). Power is produced by energy sources. If total capacity falls below total demand, the colony enters a **brownout**: all income rates halved independently of depressurization. Both penalties can stack.

Power can be exported to neighbor colonies or imported from them via trade deals (§12).

```solidity
function powerSurplus(uint256 colonyId) public view returns (int256) {
    ColonyAggregates memory a = aggregates[colonyId];
    uint256 imported = _importedPower(colonyId);
    return int256(a.powerCapacity + imported) - int256(a.powerDemand);
}

function hasPower(uint256 colonyId) public view returns (bool) {
    return powerSurplus(colonyId) >= 0;
}
```

### Resource Penalties at Claim

```solidity
// Inside claim():
uint256 o2Mod    = hasOxygen(colonyId) ? 100 : 50;  // depressurization
uint256 brownout = hasPower(colonyId)  ? 100 : 50;  // brownout
uint256 drought  = hasWater(colonyId)  ? 100 : 50;  // water crisis
// All three stack multiplicatively:
// Full crisis: 50% × 50% × 50% = 12.5% of normal income
gross = gross * o2Mod / 100 * brownout / 100 * drought / 100;
```

### Power Source Capacities and Unlock Requirements

| Source | MW | Dust | Unlock | Depth Restriction |
|---|---|---|---|---|
| Solar Array (ID 33) | 300 | None | None | **Surface only (tier 0)** |
| RTG Bank (ID 34) | 400 | None | None | Any depth |
| Nuclear Reactor (ID 35) | 5,000 | None | researchScore > 40 | Any; meltdown risk |
| Geothermal Tap (ID 36) | 1,500 | None | None | **Tier 6–7 only** |
| Microwave Receiver (ID 37) | 4,000 | None | researchScore > 70 | **Surface only** |
| Fusion Reactor (ID 38) | 10,000 | None | researchScore > 90 | Any depth |
| Waste-to-Energy (ID 30) | 400 | Low | None | Any depth |

Nuclear meltdown risk: if capacity is more than 120% utilized, a meltdown flag can be set by Chainlink VRF — destroying the reactor and creating a radiation zone that suppresses nearby habitation for N blocks.

### Power Demand by Zone Type

Each mature module draws from `powerDemand`:

| Zone | MW Draw |
|---|---|
| Habitation (per unit) | 10 / 25 / 60 (pod/module/biodome) |
| Commercial (per unit) | 30 / 80 / 200 |
| Mining (per unit) | 30 / 80 / 200 |
| Security / Emergency / Medical | 30 / 20 / 40 |
| Research Lab / Advanced Research | 30 / 50 |
| Rail / Maglev | 60 / 80 |
| Cargo Terminal / Spaceport | 100 / 150 |
| Brig | 60 |
| Radiation Shield | 40 |
| Life Support (Basic / Advanced) | 50 / 100 |

### Water Balance

Water follows the same pattern as power but affects only habitation and commercial:

```solidity
function hasWater(uint256 colonyId) public view returns (bool) {
    ColonyAggregates memory a = aggregates[colonyId];
    uint256 imported = _importedWater(colonyId);
    return (a.waterCapacity + imported) >= a.waterDemand;
}
```

Without water, habitation and commercial income rates are halved (50% modifier applied at claim, independently of brownout and depressurization). All three penalties can stack.

---

## 12. Neighbor Connectivity and Trade

### Adjacency

```solidity
function areNeighbors(uint256 colonyA, uint256 colonyB) public view returns (bool) {
    Colony storage a = colonies[colonyA];
    Colony storage b = colonies[colonyB];
    int16 dx = int16(b.surfaceX) - int16(a.surfaceX);
    int16 dy = int16(b.surfaceY) - int16(a.surfaceY);
    return (dx == 0 && (dy == 1 || dy == -1))
        || (dy == 0 && (dx == 1 || dx == -1));
}
```

Only neighbor colonies can trade. The surface map makes geography matter: a mining power-exporting colony surrounded by four hab neighbors earns from all of them.

### Tradeable Resources

All deals are `quantity / block` in exchange for `$SIM / block`. The seller must have surplus. Deals settle at claim time — no cron, no keeper needed.

| Resource | Unit | Seller Has | Buyer Gets | Direction of $SIM |
|---|---|---|---|---|
| **Power** | MW | `powerSurplus > 0` | Added to `importedPower` → counts toward `hasPower` | Buyer → Seller |
| **Oxygen** | O2 units | `oxygenSurplus > 0` | Added to `importedOxygen` → counts toward `hasOxygen` | Buyer → Seller |
| **Water** | units | `waterSurplus > 0` | Added to `importedWater` → counts toward `hasWater` | Buyer → Seller |
| **Waste disposal** | tons/block | `wasteSurplus > 0` (spare recycler capacity) | Seller absorbs buyer's waste overflow | Seller → Buyer (seller is paid to take waste) |
| **Workers** | residents | `population > localJobDemand` | Added to buyer's `importedWorkers` → boosts mining demand mod | Buyer → Seller |

### The Deal Struct

```solidity
struct Deal {
    uint256 sellerColonyId;
    uint256 buyerColonyId;
    uint8   resource;        // 0=power, 1=oxygen, 2=water, 3=waste, 4=workers
    uint32  quantity;        // units per block being traded
    uint64  simPerBlock;     // $SIM paid per block
    uint32  startBlock;
    uint32  expiryBlock;     // 0 = indefinite
    bool    active;
}

mapping(uint256 dealId => Deal) public deals;
mapping(uint256 colonyId => uint256[]) public colonyDeals; // dealIds per colony
```

### Deal Settlement at Claim

`_dealIncome(colonyId)` iterates `colonyDeals[colonyId]` — typically 0–8 deals per colony. For each active deal:

```solidity
function _dealIncome(uint256 colonyId) internal view returns (int256 net) {
    uint256[] memory ids = colonyDeals[colonyId];
    for (uint i = 0; i < ids.length; i++) {
        Deal memory d = deals[ids[i]];
        if (!d.active) continue;
        if (d.expiryBlock > 0 && block.number > d.expiryBlock) continue;

        uint256 elapsed = block.number - max(d.startBlock, colonies[colonyId].lastClaimBlock);
        uint256 payment = elapsed * d.simPerBlock;

        bool isSeller = (d.sellerColonyId == colonyId);
        net += isSeller ? int256(payment) : -int256(payment);
    }
}
```

### Resource Effects Applied via Aggregates

When a deal is accepted, the buyer's `ColonyAggregates` is immediately updated:

```solidity
function acceptDeal(uint256 dealId) external onlyBuyerGovernor(dealId) {
    Deal storage d = deals[dealId];
    require(areNeighbors(d.sellerColonyId, d.buyerColonyId), "not neighbors");

    // Validate seller has surplus
    if (d.resource == 0) require(powerSurplus(d.sellerColonyId) >= int256(d.quantity));
    if (d.resource == 1) require(oxygenSurplus(d.sellerColonyId) >= int256(d.quantity));
    if (d.resource == 2) require(waterSurplus(d.sellerColonyId) >= int256(d.quantity));
    if (d.resource == 3) require(wasteSurplus(d.sellerColonyId) >= d.quantity);
    if (d.resource == 4) require(workerSurplus(d.sellerColonyId) >= d.quantity);

    d.active     = true;
    d.startBlock = uint32(block.number);

    // Immediately apply resource to buyer's aggregates
    _snapshot(d.buyerColonyId);
    _snapshot(d.sellerColonyId);
    _applyDealToAggregates(d);
}
```

This means `hasOxygen(buyerColonyId)` returns true immediately after an oxygen deal is accepted — no waiting for claim.

### Waste Surplus

```
wasteSurplus = wasteCapPerBlock − wasteGenPerBlock − exportedWaste
```

A colony with thermal recyclers and a small population has positive waste surplus — it can accept neighbor waste. The "waste heat city" specialization: take everyone's garbage, earn $SIM/block for it. But waste processing near habitation suppresses attractiveness, so you're trading one income stream for another.

### Worker Surplus

```
workerSurplus = population − localJobDemand
localJobDemand = matureMiningCapacity + matureComCapacity
```

A hab-heavy colony with limited local mining can export its surplus population as workers to a neighbor's mining colony. The mining neighbor's `importedWorkers` is added to its effective population for `miningDemandMod`. The hab colony earns $SIM/block for the export. This creates meaningful specialization: dormitory colonies feeding mining colonies.

---

## 13. Ordinances

32 ordinances packed into one `uint256` bitmap per colony.

```solidity
mapping(uint256 colonyId => uint256) public ordinanceBitmap;

function setOrdinance(uint256 colonyId, uint8 id, bool on) external onlyGovernor(colonyId) {
    _snapshot(colonyId); // ordinance cost rate changes, bank first
    if (on) ordinanceBitmap[colonyId] |=  (1 << id);
    else    ordinanceBitmap[colonyId] &= ~(uint256(1) << id);
    _updateOrdinanceCostRate(colonyId);
}
```

Each active ordinance adds to `activeCostPerBlock` (same running rate pattern). Ordinance effects (dust modifier, tax yield modifier, demand modifier) are applied inside the view functions.

Example ordinances:
- **Water Rationing** — reduces water demand 20%, reduces hab attractiveness 5
- **Research Grants** — +researchScore, costs $SIM/block
- **Dust Suppression Protocol** — reduces dust 15%, costs $SIM/block
- **Extended Shifts** — +mining output 10%, +crime, −hab attractiveness
- **Recreation Program** — −crime, +hab attractiveness, costs $SIM/block
- **Gambling License** — +revenue, +crime

---

## 14. Disasters (Chainlink VRF)

```solidity
// Damaged zones tracked separately — avoids mutating packed zone array
mapping(uint256 colonyId => mapping(uint256 zoneIndex => bool)) public damaged;
```

When VRF fires, a random subset of zone indices is marked damaged. Damaged zones are excluded from `activeRatePerBlock` — the rate is reduced via `_snapshot` before marking damage. Repair calls `_snapshot` again and restores the rate.

**Lunar disaster types:**
- **Meteorite impact** — damages surface-tier zones (tier 0). Deeper tiers are protected.
- **Seismic event** — damages zones at random across all tiers.
- **Reactor meltdown** — triggered when nuclear capacity is >120% utilized. Destroys the reactor, irradiates surrounding sectors for N blocks.
- **Decompression breach** — damages habitation zones in a sector, temporarily reduces O2 capacity.

Depth matters for disasters: surface zones are vulnerable to meteorites but deep zones are more expensive to repair. Building critical infrastructure deep protects it from surface events.

---

## 15. Gas Cost Summary (Base, 0.005 gwei, $3k ETH)

| Action | Gas | Cost |
|---|---|---|
| Mint colony | ~80k | ~$0.12 |
| Place zone (surface) | ~35k | ~$0.05 |
| Place zone (deep, tier 7) | ~35k + $SIM excavation cost | ~$0.05 + 3× $SIM |
| Activate zone (add to rate) | ~35k–50k | ~$0.05–$0.08 |
| Upgrade zone | ~40k | ~$0.06 |
| Demolish zone | ~30k | ~$0.05 |
| **Claim (any colony size)** | **~185k** | **~$0.28** |
| Toggle ordinance | ~35k | ~$0.05 |
| Propose / accept trade deal | ~60k | ~$0.09 |
| Cancel trade deal | ~30k | ~$0.05 |
| Set tax rates | ~35k | ~$0.05 |
| Disaster VRF request | ~100k | ~$0.15 |

---

## 16. Contract Layout

```
ColonyNFT.sol          ERC-721. Colony struct, surface position, running rate accumulator,
                       treasury, tax config, snapshot logic.

ZoneRegistry.sol       Packed zone array. Place, activate, upgrade, demolish, decode.
                       Depth-tier validation (solar/geothermal restrictions).
                       Excavation cost calculation.

GameEngine.sol         Pure view functions: attractiveness, population, colonyStats,
                       researchScore, dustLevel, radiationLevel.

TradeDeals.sol         Neighbor deal creation and cancellation. dealIncome() view.
                       Supports power, oxygen, water, waste, and worker trades.

Ordinances.sol         Ordinance bitmap. Per-ordinance cost and effect tables.

DisasterOracle.sol     Chainlink VRF integration. Marks/clears damage flags.
                       Adjusts activeRatePerBlock on damage/repair.
                       Depth-aware disaster targeting (meteorites hit surface only).

SimToken.sol           ERC-20 $SIM. Minted by ClaimEngine, burned on upgrades/repairs/excavation.
```

---

## 17. What Is Not On-Chain

- **Module visuals** — the on-chain primitive is a zone placement (x, y, zoneType, level, birthBlock). What the player sees — the specific module model, size, style — is entirely client-side, derived from zone type, density, level, maturity, depth tier, and neighboring context. The contract has no concept of "module variant."
- **Tunnel / conduit / pipe rendering** — zone types 1, 2, 5 can be placed and the indexer renders them visually, but the contract treats them as no-ops. No `activateZone` call is needed or valid for these types.
- **Transit network topology** — the UI can draw rail/maglev lines between stations for visual flair. On-chain, only the sector coverage matters; actual route connectivity is not tracked.
- **Dust diffusion** — colony-level aggregate, not per-cell. The sector grid (§10) approximates proximity but does not model particle physics.
- **Depth visualization** — the client renders the colony as a cross-section or 3D underground view. On-chain, depth is just the y-coordinate of a zone placement.

---

## 18. Open Questions

- [x] ~~Grid size~~ — resolved: 128×128 colony grid, 8×8 sector grid.
- [x] ~~World map bounds~~ — resolved: auto-expanding (§1.3).
- [x] ~~Depth mechanic~~ — resolved: y-axis = depth, 8 tiers via sector rows, radiation gradient.
- [ ] Collision detection — check for overlapping (x,y) on placement, or leave to agents/UI?
- [ ] Claim rate limiting — callable anytime, or once per epoch (e.g., every 7,200 blocks)?
- [ ] Bond mechanic — borrow $SIM against `activeRatePerBlock`. Debt struct on Colony?
- [ ] Agent authorization — owner designates an agent wallet that can build/upgrade but not transfer NFT or withdraw treasury.
- [ ] Activation keeper bounty — how much $SIM to incentivize third-party `activateZone` calls?
- [ ] $SIM tokenomics — mint rate, burn sinks (upgrades, repairs, excavation), initial supply, inflation curve.
- [ ] Lava tube tiles — should some surface tiles grant a structural bonus (reduced excavation cost) baked in at mint? Same pattern as `adjacentIceTiles`. Flavor: natural caverns = pre-excavated space.
