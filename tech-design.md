# SimGame AI — Technical Design Document

> Status: In Progress | Chain: Base (L2, Coinbase)

---

## 1. Base L2 — Why and What It Changes

**Base** is Coinbase's L2 built on the OP Stack (Optimism). It is EVM-equivalent, uses ETH for gas, and posts data to Ethereum mainnet for security. Since EIP-4844 (March 2024), L1 data posting costs dropped 90%+.

**Typical Base gas costs:**
- Base fee: 0.001–0.01 gwei in normal conditions
- 50k gas tx: 0.00005–0.0005 ETH ≈ **$0.10–$1.50** at $3,000 ETH
- Block time: 2 seconds

This changes the design space significantly versus mainnet. Storage that would cost $100 on mainnet costs $0.10–$1 on Base. We can afford per-epoch settlement, richer on-chain stats, and frequent agent transactions.

All cost estimates in this document use **0.005 gwei and $3,000 ETH** as the baseline.

---

## 2. City NFT — ERC-721

Each city is a unique ERC-721 token. Uniqueness comes from the terrain seed — no two cities have the same landscape.

**No sub-parcel tokens.** The city NFT represents the entire 1024×1024 grid. Owning the NFT = owning every square.

### 2.1 Core Storage Per City

```solidity
struct City {
    uint32  terrainSeed;       // deterministic terrain generation seed
    uint32  mintedBlock;       // block city was minted (era tracking)
    uint16  gridSize;          // always 1024 for v1
    address mayorAgent;        // AI agent wallet authorized to zone/budget
    uint256 treasury;          // $SIM balance held by city
}

mapping(uint256 cityId => City) public cities;
```

Mint cost: 3 SSTORE ops = ~66,000 gas ≈ **$0.10**.

---

## 3. Terrain — Seed-Based, No Off-Chain Storage

### 3.1 The Problem

A 1024×1024 terrain at 7 bits/cell (1 bit land/water + 6 bits elevation) = ~917KB. On-chain storage at 32 bytes/slot costs ~640,000 SSTORE ops = hundreds of millions of gas. Arweave introduces off-chain dependency. Both are unacceptable.

### 3.2 Solution: Deterministic Seed

Store a single `uint32 terrainSeed` per city. Terrain for any cell is computed on demand:

```solidity
function cellType(uint32 seed, uint16 x, uint16 y)
    public pure returns (bool isLand, uint8 elevation)
{
    bytes32 h = keccak256(abi.encodePacked(seed, x, y));
    uint256 v = uint256(h);
    isLand    = (v & 0xFF) > WATER_THRESHOLD;         // ~30% water
    elevation = isLand ? uint8((v >> 8) & 0x3F) : 0; // 0–63 if land
}
```

- Any smart contract can verify terrain for a given cell with one function call.
- Clients generate the full 1024×1024 map locally from the seed.
- Terrain is immutable — stored once at mint.
- **Cost: 0 extra storage vs. no terrain at all.**

### 3.3 Terraforming (Mechanic + Practical Escape Valve)

Agents can modify individual cells by spending $SIM. Modifications stored as a sparse delta mapping:

```solidity
// packed: int8 elevDelta (bits 0-7), uint8 typeOverride (bits 8-15)
mapping(uint256 cityId => mapping(uint20 coord => uint16 terrainDelta)) public terraformDeltas;
```

Terraform rules:
- Max elevation delta: ±8 per cell
- Water → land conversion: costs 500 $SIM/cell (major modification)
- Land → water channel: costs 200 $SIM/cell
- Raise/lower elevation: costs 50 $SIM/cell/tier

This gives agents real strategic control (flatten hills to expand buildable land, create water access for pumps) while keeping cost proportional to impact.

**Terraform cost on Base:** 1 SSTORE per cell modified ≈ 22k gas ≈ **$0.03/cell**.

---

## 4. Zone Storage — Rectangle Command Log

### 4.1 The Problem with Per-Cell Storage

Storing zone type + build data per cell:
- 1M cells × 1 SSTORE each = 22B gas for a fully developed city
- Even at Base prices that's hundreds of thousands of dollars — absurd

### 4.2 Rectangle Command Log

Every zone operation stores a single `uint256` packed with the rectangle bounds, zone type, and placement block. **Any size rectangle = one SSTORE.**

```
bit layout (256 bits total):
 0– 9:  x1         (10 bits, 0–1023)
10–19:  y1         (10 bits)
20–29:  x2         (10 bits)
30–39:  y2         (10 bits)
40–43:  zoneType   (4 bits, see table §4.3)
44–83:  placedBlock (40 bits, ~34B blocks of headroom = ~2,200 years at Base 2s/block)
84–87:  flags      (4 bits: upgraded, damaged, demolished, reserved)
88–255: reserved
```

```solidity
mapping(uint256 cityId => uint256[]) public zoneOps;
```

**Gas cost for any zone operation:** ~50,000 gas ≈ **$0.08**
This is the same cost whether you zone 1 cell or 500×500 cells.

### 4.3 Zone Types (4 bits = 16 types)

| ID | Name | Matures (Base blocks) | Notes |
|---|---|---|---|
| 0 | EMPTY (demolish) | — | Clears underlying op |
| 1 | ROAD | 600 (~20 min) | Infrastructure |
| 2 | POWER_LINE | 300 (~10 min) | Infrastructure |
| 3 | RESIDENTIAL_LIGHT | 43,200 (~1 day) | |
| 4 | RESIDENTIAL_MEDIUM | 129,600 (~3 days) | Requires demand > 50, LV > 40 |
| 5 | RESIDENTIAL_DENSE | 259,200 (~6 days) | Requires demand > 100, LV > 70 |
| 6 | COMMERCIAL_LIGHT | 43,200 | |
| 7 | COMMERCIAL_MEDIUM | 129,600 | |
| 8 | COMMERCIAL_DENSE | 259,200 | |
| 9 | INDUSTRIAL_LIGHT | 86,400 (~2 days) | Low pollution |
| 10 | INDUSTRIAL_MEDIUM | 172,800 (~4 days) | Med pollution |
| 11 | INDUSTRIAL_DENSE | 345,600 (~8 days) | High pollution |
| 12 | INFRASTRUCTURE | 43,200 | Power plants, water, transit, services |
| 13 | LANDFILL | 21,600 (~12 hrs) | Fills over time |
| 14 | PARK | 21,600 | Aura + land value |
| 15 | RESERVED | — | Future use |

Infrastructure zone ops carry a `subType` in the reserved bits (extended to 8 bits in a second encoding variant) to distinguish power plant type, water pump vs. tower, school vs. hospital, etc.

### 4.4 Overwrite Semantics

Later ops override earlier ops for overlapping cells. The canonical state of any cell is the **most recent** op covering it. The indexer maintains a current-state map. The contract trusts the log order.

To demolish: zone `EMPTY` over the area.

### 4.5 Batching

Multiple zone ops in a single transaction save the ~21k base tx overhead:

```solidity
function zoneRectBatch(
    uint256 cityId,
    ZoneOp[] calldata ops
) external onlyMayor(cityId) withCheckpoint(cityId) {
    for (uint i = 0; i < ops.length; i++) {
        _addZoneOp(cityId, ops[i]);
    }
}
```

**10 zone ops in one tx:** ~250,000 gas ≈ **$0.38** (vs. 10 separate txs at $0.80).

---

## 5. Build States — Automatic, No Stored State

Build state is **never written to storage**. It is derived on any read from `block.number - placedBlock`.

```solidity
enum ZoneState { STALLED, FOUNDATION, CONSTRUCTION, COMPLETE, MATURE, UPGRADE_ELIGIBLE, DAMAGED }

uint256[16] public MATURATION_BLOCKS; // set at deploy for each zone type

function getZoneState(uint256 cityId, uint256 opIndex)
    public view returns (ZoneState)
{
    uint256 op = zoneOps[cityId][opIndex];
    uint8   zoneType    = uint8((op >> 40) & 0xF);
    uint40  placedBlock = uint40((op >> 44));
    uint8   flags       = uint8((op >> 84) & 0xF);

    if (flags & DAMAGED_FLAG != 0) return ZoneState.DAMAGED;

    // Infrastructure zones never stall
    bool infra = (zoneType == ZONE_ROAD || zoneType == ZONE_POWER_LINE
               || zoneType == ZONE_INFRASTRUCTURE);

    if (!infra) {
        // Non-infra stalls without city-level power
        if (!_cityHasPower(cityId))  return ZoneState.STALLED;
        // Residential/commercial stalls without water (except industrial)
        if (zoneType <= ZONE_COMMERCIAL_DENSE && !_cityHasWater(cityId))
            return ZoneState.STALLED;
    }

    uint256 age    = block.number - placedBlock;
    uint256 mature = MATURATION_BLOCKS[zoneType];

    if (age < mature / 4)  return ZoneState.FOUNDATION;
    if (age < mature / 2)  return ZoneState.CONSTRUCTION;
    if (age < mature)      return ZoneState.COMPLETE;
    if (age < mature * 3)  return ZoneState.MATURE;
    return ZoneState.UPGRADE_ELIGIBLE;
}
```

**This is a pure view function — zero gas on read, never writes storage.** Build states update passively as blocks pass. No transactions required for a zone to mature.

### 5.1 Partial Output During Construction

Zones in CONSTRUCTION produce 50% of their mature output. COMPLETE produces 75%. MATURE produces 100%. UPGRADE_ELIGIBLE produces 110% (bonus for long-standing buildings).

---

## 6. Aggregate City Stats — Auto-Checkpoint

Per-cell state is derivable from the zone op log + `block.number`. On-chain, we maintain aggregate counters used for milestone checks, population, and reward calculations. These must stay current without manual triggers.

### 6.1 The Auto-Checkpoint Modifier

Every write function that modifies city state runs `_checkpoint(cityId)` before executing:

```solidity
modifier withCheckpoint(uint256 cityId) {
    _checkpoint(cityId);
    _;
    _syncAttractiveness(cityId);
}

function _checkpoint(uint256 cityId) internal {
    CityStats storage s = stats[cityId];
    uint256 scanFrom = s.lastCheckpointOpIndex;
    uint256 total    = zoneOps[cityId].length;
    uint256 limit    = scanFrom + MAX_OPS_PER_CHECKPOINT; // e.g., 50
    if (limit > total) limit = total;

    for (uint256 i = scanFrom; i < limit; i++) {
        uint256 op         = zoneOps[cityId][i];
        uint8   zoneType   = uint8((op >> 40) & 0xF);
        uint40  placed     = uint40((op >> 44));
        uint256 matureAt   = placed + MATURATION_BLOCKS[zoneType];
        uint32  cells      = _rectArea(op); // (x2-x1+1) * (y2-y1+1)

        if (block.number >= matureAt) {
            _incrementStats(s, zoneType, cells);
        }
    }
    s.lastCheckpointOpIndex = uint32(limit);
    s.lastCheckpointBlock   = uint48(block.number);
}
```

**Key properties:**
- Runs automatically on every agent write transaction — no separate call needed.
- Bounded scan (MAX_OPS_PER_CHECKPOINT) prevents gas explosion on large cities.
- Large cities catch up over multiple sequential transactions (normal agent cadence).
- For milestone claims, `forceCheckpoint(cityId)` can be called to complete any remaining scan at cost to the caller. This is the ONE manual trigger — for milestone claiming only.

```solidity
struct CityStats {
    uint32 matureResLight;    uint32 matureResMed;    uint32 matureResDense;
    uint32 matureComLight;    uint32 matureComMed;    uint32 matureComDense;
    uint32 matureIndLight;    uint32 matureIndMed;    uint32 matureIndDense;
    uint32 matureRoadCells;   uint32 maturePowerCells;
    uint32 matureWaterCells;  uint32 matureServiceCells;
    uint32 matureParkCells;   uint32 matureLandfillCells;
    uint32 lastCheckpointOpIndex;
    uint48 lastCheckpointBlock;
}
```

Packed into 4 uint256 slots → 4 SSTORE writes per checkpoint update ≈ 88k gas ≈ **$0.13**.

---

## 7. Derived City Metrics — Pure View Functions

All game calculations are pure view functions. They read `CityStats` + zone op log + `block.number`. No additional storage required.

### 7.1 Power and Water Status

```solidity
function _cityHasPower(uint256 cityId) internal view returns (bool) {
    // True if at least one INFRASTRUCTURE zone of subType POWER_PLANT is MATURE
    return stats[cityId].matureServiceCells > 0 /* simplified: infra includes power */
        && _hasMaturePowerPlant(cityId);
}
```

v1 simplification: city has power = has ≥1 mature power plant + power line coverage > 10%.

### 7.2 Population Capacity

```solidity
function residentialCapacity(uint256 cityId) public view returns (uint256) {
    CityStats memory s = stats[cityId];
    return s.matureResLight  * 100
         + s.matureResMed    * 300
         + s.matureResDense  * 700;
}
```

### 7.3 Education Score

```solidity
function educationScore(uint256 cityId) public view returns (uint256) {
    // Simplified: service cells include school/college/library subtypes
    uint256 pop = residentialCapacity(cityId);
    if (pop == 0) return 0;
    return min(100, stats[cityId].matureServiceCells * EDUCATION_PER_SERVICE * 100 / pop);
}
```

### 7.4 Pollution

```solidity
function pollutionScore(uint256 cityId) public view returns (uint256) {
    CityStats memory s = stats[cityId];
    uint256 edu = educationScore(cityId);

    uint256 industrialPollution = (
        s.matureIndLight  * 10 +
        s.matureIndMed    * 30 +
        s.matureIndDense  * 80
    ) * max(15, 100 - edu) / 100;

    uint256 trafficPollution = _trafficScore(cityId) / 5;
    uint256 parkReduction    = s.matureParkCells * 2;

    return max(0, industrialPollution + trafficPollution - parkReduction);
}
```

### 7.5 Land Value

```solidity
function landValue(uint256 cityId) public view returns (uint256) {
    CityStats memory s = stats[cityId];
    uint256 base = 50;
    uint256 bonus =
        (_cityHasWater(cityId)   ? 20 : 0) +
        (s.matureParkCells       *  3)      +
        (s.matureServiceCells    *  5)      ;
    uint256 penalty =
        (pollutionScore(cityId)  / 2)       +
        (_crimeScore(cityId)     * 3)       +
        (s.matureLandfillCells   * 4)       ;
    uint256 ord = _ordinanceLandValueBonus(cityId);
    return base + bonus + ord > penalty ? base + bonus + ord - penalty : 0;
}
```

### 7.6 Attractiveness Score

```solidity
function attractiveness(uint256 cityId) public view returns (uint256) {
    if (!_cityHasPower(cityId)) return 0;

    CityStats memory s = stats[cityId];
    uint256 score = residentialCapacity(cityId) / 10
        + (s.matureComLight + s.matureComMed * 2 + s.matureComDense * 4) * 5
        + s.matureServiceCells * 8
        + s.matureParkCells    * 3
        + (_cityHasWater(cityId) ? 50 : 0)
        - pollutionScore(cityId)
        - _crimeScore(cityId) * 2;

    return score;
}
```

### 7.7 Population (Inter-City Competition)

```solidity
uint256 public constant BASELINE_ATTRACTIVENESS = 500;

function population(uint256 cityId) public view returns (uint256) {
    uint256 cap   = residentialCapacity(cityId);
    uint256 score = attractiveness(cityId);
    if (cap == 0 || score == 0) return 0;

    // Occupancy scales with attractiveness vs. global baseline
    // Above baseline = above 50% occupancy; below = below 50%
    // Capped at 100%
    uint256 occupancy = score * 5e17 / BASELINE_ATTRACTIVENESS; // 0.5 at baseline
    if (occupancy > 1e18) occupancy = 1e18;

    return cap * occupancy / 1e18;
}
```

Each city registers its attractiveness in `GameEngine` when its stats are checkpointed (auto via modifier). No external sync call needed — attractiveness is always current because `_syncAttractiveness(cityId)` runs after every write:

```solidity
function _syncAttractiveness(uint256 cityId) internal {
    uint256 newScore = attractiveness(cityId);
    GameEngine(gameEngine).updateCityScore(cityId, newScore);
}
```

`GameEngine.updateCityScore` does a delta update on `totalAttractiveness` — one SSTORE. **This is automatic on every agent transaction. No manual trigger.**

---

## 8. Budget & Epoch Settlement

### 8.1 Budget Config (Packed)

```solidity
struct Budget {
    uint8 resTaxRate;          // 0–20 (percent)
    uint8 comTaxRate;
    uint8 indTaxRate;
    uint8 policeFunding;       // 50–150 (scaled: 100 = 100%)
    uint8 fireFunding;
    uint8 healthFunding;
    uint8 educationFunding;
    uint8 transitFunding;
    // remaining 192 bits: bond data
    uint64 activeBondAmount;
    uint32 bondRepayPerEpoch;
    uint32 bondEpochsRemaining;
}
// Packed: fits in 2 uint256 slots
```

### 8.2 Epoch Definition

```solidity
uint256 public constant EPOCH_BLOCKS = 7_200; // 4 hours at 2s/block on Base
```

### 8.3 Lazy Revenue Settlement

Revenue does not auto-update every epoch (that would require a keeper). Instead it settles lazily when `settleEpochs(cityId)` is called. The function computes all elapsed epochs in a loop:

```solidity
function settleEpochs(uint256 cityId) external {
    uint256 lastSettled = lastEpochSettled[cityId];
    uint256 currentEpoch = block.number / EPOCH_BLOCKS;
    uint256 epochs = currentEpoch - lastSettled;
    require(epochs > 0, "nothing to settle");

    Budget memory b = budgets[cityId];
    CityStats memory s = stats[cityId];

    // Per-epoch revenue
    uint256 resCells = s.matureResLight + s.matureResMed * 3 + s.matureResDense * 7;
    uint256 comCells = s.matureComLight + s.matureComMed * 3 + s.matureComDense * 7;
    uint256 indCells = s.matureIndLight + s.matureIndMed * 2 + s.matureIndDense * 4;

    uint256 taxIncome = resCells * b.resTaxRate * RES_VALUE_PER_CELL / 100
                      + comCells * b.comTaxRate * COM_VALUE_PER_CELL / 100
                      + indCells * b.indTaxRate * IND_VALUE_PER_CELL / 100;

    uint256 opCosts = _operatingCosts(s, b);

    // Garbage balance
    uint256 garbagePerEpoch = population(cityId) * GARBAGE_PER_RESIDENT;
    uint256 garbageCapacity = _garbageCapacity(s);
    uint256 overflow        = garbagePerEpoch > garbageCapacity
                              ? garbagePerEpoch - garbageCapacity : 0;

    int256 epochNet = int256(taxIncome) - int256(opCosts) - int256(b.bondRepayPerEpoch);
    int256 totalNet = epochNet * int256(epochs);

    // Update treasury
    if (totalNet > 0) {
        cities[cityId].treasury += uint256(totalNet);
        // Surplus → mint $SIM reward
        SimToken(simToken).mint(ownerOf(cityId), uint256(totalNet) * SURPLUS_REWARD_RATE / 1e18);
    } else {
        uint256 deficit = uint256(-totalNet);
        require(cities[cityId].treasury >= deficit, "city bankrupt");
        cities[cityId].treasury -= deficit;
    }

    // Garbage overflow aura penalty persists in stats
    if (overflow > 0) _applyGarbageOverflow(cityId, overflow * epochs);

    lastEpochSettled[cityId] = currentEpoch;

    // Small settlement bounty for caller
    SimToken(simToken).mint(msg.sender, SETTLEMENT_BOUNTY);
}
```

**The small settlement bounty ensures `settleEpochs` gets called regularly by anyone.** The city owner is also directly incentivized (treasury grows with surplus). This is a financial action, not a simulation trigger.

---

## 9. Ordinances — Packed Bitmap

All ordinances are toggles stored as a single `uint256` bitmap per city:

```solidity
mapping(uint256 cityId => uint256) public ordinances;

// Ordinance IDs (0–31):
// 0: Water Conservation     1: Power Conservation   2: Stairwell Lighting
// 3: Mandatory Water Meters  4: Free Clinics         5: Junior Sports
// 6: Nuclear Free Zone       7: Pro-Reading          8: Public Smoking Ban
// 9: Neighborhood Watch      10: Smoke Detectors     11: Youth Curfew
// 12: Crossing Guards        13: Leaf Burning Ban     14: Landfill Gas Recovery
// 15: Industrial Waste Tax    16: Clean Air           17: Trash Presort
// 18: Lawn Chemical Ban       19: Backyard Composting 20: Paper Reduction
// 21: Tire Recycling          22: Industrial Pollutant Fee
// 23: Electronics Tax         24: Homeless Shelters   25: Tourist Promotion
// 26: Conservation Corps      27: Alt Day Driving     28: Subsidized Transit
// 29: Carpool Incentive       30: Parking Fines       31: Legalized Gambling

function setOrdinance(uint256 cityId, uint8 ordId, bool enabled)
    external onlyMayor(cityId)
{
    if (enabled) {
        ordinances[cityId] |= (1 << ordId);
    } else {
        ordinances[cityId] &= ~(uint256(1) << ordId);
    }
}

function hasOrdinance(uint256 cityId, uint8 ordId) public view returns (bool) {
    return (ordinances[cityId] >> ordId) & 1 == 1;
}
```

Ordinance effects are applied as multipliers in the derived metric view functions (`pollutionScore`, `landValue`, `population`, etc.). **All 32 ordinances fit in 1 uint256 slot.**

---

## 10. Inter-City Neighbor Deals

### 10.1 Neighbor Slots

```solidity
// Each city can register up to 4 neighbors
mapping(uint256 cityId => uint256[4]) public neighbors;
```

### 10.2 Deal Structure

```solidity
struct Deal {
    uint256 fromCity;
    uint256 toCity;
    uint8   dealType;     // 0: power, 1: water, 2: garbage, 3: toxic waste
    uint128 amountPerEpoch;  // units of capacity exported
    uint128 simPerEpoch;     // $SIM paid per epoch
    uint32  startEpoch;
    uint32  durationEpochs;  // 0 = indefinite
    bool    active;
}

mapping(uint256 dealId => Deal) public deals;
```

### 10.3 Settlement

Deals settle during `settleEpochs`. The exporting city's capacity is reduced; the importing city gets the benefit. $SIM transfers between city treasuries. No keeper needed — deal effects are applied whenever either city settles epochs.

---

## 11. Disasters — Chainlink VRF

```solidity
function requestDisaster(uint256 cityId) external returns (uint256 requestId);

// VRF callback
function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords)
    external override
{
    uint256 cityId = requestToCityId[requestId];
    uint256 rand   = randomWords[0];

    // Probability gates (based on city risk stats)
    // Low fire coverage → fire event
    // High industrial density → industrial accident
    // Fault line seed match → earthquake
    // Random rare event → UFO / tornado / riot
    _resolveDisaster(cityId, rand);
}

function _resolveDisaster(uint256 cityId, uint256 rand) internal {
    // Pick random zone ops in the city and set DAMAGED flag
    uint256 opCount = zoneOps[cityId].length;
    uint256 affected = 1 + (rand % MAX_DISASTER_RADIUS);
    for (uint i = 0; i < affected; i++) {
        uint256 opIdx = (rand >> (i * 8)) % opCount;
        zoneOps[cityId][opIdx] |= (uint256(DAMAGED_FLAG) << 84);
    }
    emit DisasterOccurred(cityId, DisasterType(rand % 5));
}
```

---

## 12. Smart Contract Architecture

```
CityNFT.sol
  ├─ ERC-721
  ├─ terrain seed + terraform deltas
  ├─ zoneOps log
  ├─ CityStats (auto-checkpointed)
  ├─ Budget config
  ├─ Ordinance bitmap
  └─ Treasury ($SIM balance)

GameEngine.sol
  ├─ attractiveness() view
  ├─ population() view
  ├─ landValue() view
  ├─ pollutionScore() view
  ├─ rciDemand() view
  ├─ cityAttractiveness mapping (updated by CityNFT on every write)
  ├─ totalAttractiveness (global sum)
  └─ milestone verification

EpochSettler.sol
  ├─ settleEpochs(cityId)
  ├─ neighbor deal settlement
  └─ bounty distribution

NeighborDeals.sol
  ├─ proposeDeal / acceptDeal / cancelDeal
  └─ deal registry

DisasterOracle.sol
  ├─ Chainlink VRF integration
  └─ disaster resolution

SimToken.sol
  └─ ERC-20 $SIM

CityMarket.sol
  └─ list/buy/sell city NFTs
```

### 12.1 Contract Interaction Flow

```
Agent Wallet
    │
    ├─ zoneRectBatch(cityId, ops[])
    │     │ withCheckpoint(cityId):
    │     │   _checkpoint() → updates CityStats
    │     │   _syncAttractiveness() → updates GameEngine.cityAttractiveness
    │     └─ adds ops to zoneOps[cityId]
    │
    ├─ setTaxRates(cityId, budget)
    │     └─ withCheckpoint(cityId) [same auto-update]
    │
    ├─ setOrdinance(cityId, ordId, enabled)
    │     └─ withCheckpoint(cityId)
    │
    ├─ settleEpochs(cityId)         ← anyone can call (bounty incentive)
    │     └─ computes revenue/costs for all elapsed epochs
    │
    └─ claimMilestone(cityId, milestone)
          └─ forceCheckpoint + GameEngine.population() >= milestone.threshold
             → SimToken.mint(owner, reward)
```

---

## 13. Gas Cost Reference Table (Base L2, 0.005 gwei, $3k ETH)

| Operation | Gas (est.) | Cost |
|---|---|---|
| Mint city | ~80,000 | ~$0.12 |
| Zone 1 rectangle | ~50,000 | ~$0.08 |
| Zone 10 rectangles (batch) | ~200,000 | ~$0.30 |
| Terraform 1 cell | ~30,000 | ~$0.05 |
| Set tax rates / budget | ~40,000 | ~$0.06 |
| Toggle ordinance | ~30,000 | ~$0.05 |
| Settle epochs (10 epochs) | ~120,000 | ~$0.18 |
| Force checkpoint (50 ops) | ~150,000 | ~$0.23 |
| Claim milestone | ~100,000 | ~$0.15 |
| Propose neighbor deal | ~60,000 | ~$0.09 |
| Disaster VRF request | ~100,000 | ~$0.15 |

An active AI agent making 10 decisions/day costs approximately **$0.30–$3.00/day** in gas.

---

## 14. Indexer

The Graph (or a custom indexer) listens to `ZoneOpAdded` events and maintains current cell state per city:

```
For each ZoneOpAdded(cityId, op):
  decode op → (x1, y1, x2, y2, zoneType, placedBlock)
  For each cell (x, y) in rect:
    currentState[cityId][x][y] = { zoneType, placedBlock }
```

The indexer also computes build state per cell on query:
```
buildState = getZoneState(zoneType, placedBlock, currentBlock)
```

This gives the frontend a queryable per-cell map without on-chain per-cell storage. **All on-chain storage remains aggregate and cheap. Per-cell detail lives in the indexer.**

---

## 15. Open Technical Questions

- [ ] **MAX_OPS_PER_CHECKPOINT** — what bound prevents gas exhaustion while keeping stats current? 50? 100? Needs profiling.
- [ ] **Infrastructure subtype encoding** — extend zone type to 8 bits (split with a second packed word) or use a separate `infraType` mapping per op index?
- [ ] **Garbage overflow persistence** — aura penalty stored as a time-decaying flag or recomputed each epoch?
- [ ] **Bond defaulting** — what happens on bankruptcy? City goes into receivership (anyone can take it over)?
- [ ] **Earthquake fault lines** — derived from terrain seed (certain seeds = fault line cities)? Adds strategic value to city selection.
- [ ] **Upgrade mechanic** — `UPGRADE_ELIGIBLE` triggers a new zone op with a "V2" zoneType, or modify flags in the existing op?
- [ ] **City merge** — adjacent city NFTs share a border. Define border adjacency via tokenId grid placement.
- [ ] **Chainlink VRF subscription** — per-city subscription or shared pool subscription with city-specific entropy?
