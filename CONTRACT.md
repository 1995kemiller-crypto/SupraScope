# SupraScope — Move Contract Specification

> Version 0.1 · Supra Network · BTC initial implementation

---

## Overview

The SupraScope Move contract is the canonical onchain truth. It holds the `ObservatoryState` for each asset — a first-class resource on the Supra blockchain, verifiable by any contract or consumer.

The contract does not run the physics engine. It holds the result of the physics engine, signed and submitted by the authorized offchain compute node each epoch.

**What lives onchain:**
- Current state vector per asset (six signed quantities)
- State history ring buffer (last N epochs)
- Nominal state per asset
- Topology event log
- Asset registry

**What lives offchain:**
- Physics computation (ODE solver, Hawkes estimator)
- Data ingestion (Supra oracle, Binance, social feeds)
- Normalization and fingerprinting
- Renderer and API layer

---

## Module Structure

```
suprascope/
  sources/
    observatory.move        — main contract
    state.move              — ObservatoryState struct and methods
    history.move            — ring buffer implementation
    topology.move           — topology event types and log
    registry.move           — asset registry
    errors.move             — error codes
  tests/
    observatory_tests.move
  Move.toml
```

---

## Core Structs

### `ObservatoryState`

The primary resource. One instance per asset, stored in the global state table.

```move
struct ObservatoryState has key, store {
    // Asset identifier
    asset_id: vector<u8>,           // e.g. b"BTC"

    // Current epoch
    epoch: u64,
    timestamp: u64,                 // Unix milliseconds

    // Six physics quantities — signed fixed-point
    // Stored as i64, scaled by 1000 (so 4.2 = 4200, -3.8 = -3800)
    // Range: -10000 to 10000 (representing -10.0 to +10.0)
    inertia:     i64,
    tension:     i64,
    compression: i64,
    expansion:   i64,
    turbulence:  i64,
    memory:      i64,

    // Memory layers (short, mid, long) — same signed fixed-point
    short_inertia:     i64,
    short_tension:     i64,
    short_compression: i64,
    short_expansion:   i64,
    short_turbulence:  i64,
    short_memory:      i64,

    mid_inertia:     i64,
    mid_tension:     i64,
    mid_compression: i64,
    mid_expansion:   i64,
    mid_turbulence:  i64,
    mid_memory:      i64,

    long_inertia:     i64,
    long_tension:     i64,
    long_compression: i64,
    long_expansion:   i64,
    long_turbulence:  i64,
    long_memory:      i64,

    // Nominal state
    nominal_inertia:     i64,
    nominal_tension:     i64,
    nominal_compression: i64,
    nominal_expansion:   i64,
    nominal_turbulence:  i64,
    nominal_memory:      i64,
    nominal_confidence:  u64,       // 0–10000 (0.0–1.0 scaled)

    // Deviation from nominal
    deviation_magnitude: u64,       // unsigned, 0–14142 (√2 * 10000)
    deviation_velocity:  i64,       // signed — accelerating or decelerating

    // Energy level
    energy: u64,                    // 0–10000

    // Topology
    active_topology_event: u8,      // 0=none 1=rupture 2=fold 3=collapse 4=bloom
    last_topology_epoch: u64,
    last_topology_type: u8,

    // Regime label
    regime: u8,                     // see regime enum below

    // Submitter
    submitted_by: address,
}
```

### `StateHistory`

Ring buffer of historical states per asset.

```move
struct StateHistory has key, store {
    asset_id: vector<u8>,
    capacity: u64,                  // 11520 = 2 days at 15s epochs
    head: u64,                      // current write position
    epochs: vector<u64>,
    timestamps: vector<u64>,
    // Compressed state vectors — only primary six quantities
    // to keep storage cost manageable
    inertia_history:     vector<i64>,
    tension_history:     vector<i64>,
    compression_history: vector<i64>,
    expansion_history:   vector<i64>,
    turbulence_history:  vector<i64>,
    memory_history:      vector<i64>,
    energy_history:      vector<u64>,
}
```

### `TopologyEvent`

Immutable record of a topology event. Appended to the event log, never modified.

```move
struct TopologyEvent has store, drop {
    asset_id:   vector<u8>,
    epoch:      u64,
    timestamp:  u64,
    event_type: u8,                 // 1=rupture 2=fold 3=collapse 4=bloom
    magnitude:  u64,                // 0–10000
    streams_involved: u8,           // bitmask: bit0=price bit1=micro bit2=deriv bit3=onchain bit4=social bit5=macro
}
```

### `AssetRegistry`

Registry of all active assets with their configuration.

```move
struct AssetRegistry has key {
    assets: vector<AssetConfig>,
}

struct AssetConfig has store, drop {
    asset_id:            vector<u8>,
    symbol:              vector<u8>,
    active:              bool,
    epoch_length_ms:     u64,
    authorized_submitter: address,
    created_epoch:       u64,
    nominal_confidence:  u64,
}
```

---

## Entry Functions

### `update_state`

Called by the authorized offchain compute node each epoch. Writes the new state, updates history, fires events.

```move
public entry fun update_state(
    submitter: &signer,
    asset_id: vector<u8>,
    epoch: u64,
    timestamp: u64,

    // Six quantities (scaled i64)
    inertia: i64,
    tension: i64,
    compression: i64,
    expansion: i64,
    turbulence: i64,
    memory_val: i64,

    // Memory layers (18 values)
    short_layer: vector<i64>,       // length 6
    mid_layer:   vector<i64>,       // length 6
    long_layer:  vector<i64>,       // length 6

    // Nominal state (7 values including confidence)
    nominal:     vector<i64>,       // length 6
    nominal_confidence: u64,

    // Derived quantities
    deviation_magnitude: u64,
    deviation_velocity:  i64,
    energy: u64,

    // Topology
    active_topology_event: u8,
    topology_magnitude:    u64,
    streams_involved:      u8,

    // Regime
    regime: u8,
) acquires ObservatoryState, StateHistory
```

**Validations:**
- `submitter` must match `authorized_submitter` for this asset
- `epoch` must be exactly `current_epoch + 1`
- All quantity values must be in range [−10000, 10000]
- `timestamp` must be within acceptable drift of chain time
- `nominal_confidence` must be in range [0, 10000]

**Effects:**
1. Updates `ObservatoryState` with new values
2. Appends current state to `StateHistory` ring buffer
3. If `active_topology_event != 0`, appends to topology event log
4. Emits `StateUpdated` event
5. If topology event, emits `TopologyEventFired` event

---

### `register_asset`

Register a new asset for tracking. Admin only.

```move
public entry fun register_asset(
    admin: &signer,
    asset_id: vector<u8>,
    symbol: vector<u8>,
    epoch_length_ms: u64,
    authorized_submitter: address,
) acquires AssetRegistry
```

---

### `update_authorized_submitter`

Rotate the authorized submitter address for an asset. Admin only.

```move
public entry fun update_authorized_submitter(
    admin: &signer,
    asset_id: vector<u8>,
    new_submitter: address,
) acquires AssetRegistry
```

---

## View Functions

### `get_current_state`

```move
#[view]
public fun get_current_state(
    asset_id: vector<u8>
): ObservatoryState acquires ObservatoryState
```

### `get_state_at_epoch`

```move
#[view]
public fun get_state_at_epoch(
    asset_id: vector<u8>,
    epoch: u64
): (i64, i64, i64, i64, i64, i64, u64) acquires StateHistory
// returns (inertia, tension, compression, expansion, turbulence, memory, energy)
```

### `get_history_range`

```move
#[view]
public fun get_history_range(
    asset_id: vector<u8>,
    from_epoch: u64,
    to_epoch: u64
): (vector<u64>, vector<i64>, vector<i64>, vector<i64>, 
    vector<i64>, vector<i64>, vector<i64>) acquires StateHistory
// returns (epochs, inertia[], tension[], compression[], expansion[], turbulence[], memory[])
```

### `get_topology_events`

```move
#[view]
public fun get_topology_events(
    asset_id: vector<u8>,
    from_epoch: u64,
    to_epoch: u64
): vector<TopologyEvent> acquires ObservatoryState
```

### `get_nominal_state`

```move
#[view]
public fun get_nominal_state(
    asset_id: vector<u8>
): (i64, i64, i64, i64, i64, i64, u64) acquires ObservatoryState
// returns (inertia, tension, compression, expansion, turbulence, memory, confidence)
```

---

## Events

```move
struct StateUpdated has drop, store {
    asset_id:  vector<u8>,
    epoch:     u64,
    timestamp: u64,
    energy:    u64,
    regime:    u8,
}

struct TopologyEventFired has drop, store {
    asset_id:        vector<u8>,
    epoch:           u64,
    timestamp:       u64,
    event_type:      u8,
    magnitude:       u64,
    streams_involved: u8,
}

struct AssetRegistered has drop, store {
    asset_id: vector<u8>,
    symbol:   vector<u8>,
    epoch:    u64,
}
```

---

## Regime Enum

```move
const REGIME_NOMINAL:              u8 = 0;
const REGIME_EXPANDING_COHERENT:   u8 = 1;
const REGIME_EXPANDING_FRAGMENTED: u8 = 2;
const REGIME_COMPRESSING:          u8 = 3;
const REGIME_HIGH_TENSION:         u8 = 4;
const REGIME_TURBULENT:            u8 = 5;
const REGIME_EERIE_COHERENT:       u8 = 6;
const REGIME_MEMORY_ACTIVE:        u8 = 7;
const REGIME_NOVEL_TERRITORY:      u8 = 8;
const REGIME_PRE_TOPOLOGY:         u8 = 9;
```

---

## Topology Event Enum

```move
const TOPOLOGY_NONE:     u8 = 0;
const TOPOLOGY_RUPTURE:  u8 = 1;
const TOPOLOGY_FOLD:     u8 = 2;
const TOPOLOGY_COLLAPSE: u8 = 3;
const TOPOLOGY_BLOOM:    u8 = 4;
```

---

## Error Codes

```move
const E_NOT_AUTHORIZED:       u64 = 1;
const E_WRONG_EPOCH:          u64 = 2;
const E_VALUE_OUT_OF_RANGE:   u64 = 3;
const E_TIMESTAMP_DRIFT:      u64 = 4;
const E_ASSET_NOT_FOUND:      u64 = 5;
const E_ASSET_ALREADY_EXISTS: u64 = 6;
const E_INVALID_LAYER_LENGTH: u64 = 7;
const E_HISTORY_EMPTY:        u64 = 8;
const E_EPOCH_NOT_IN_HISTORY: u64 = 9;
```

---

## Composability

Any contract on Supra can read SupraScope state directly:

```move
use suprascope::observatory;

// Read current BTC state
let (inertia, tension, compression, expansion, turbulence, memory_val, energy) =
    observatory::get_current_state(b"BTC");

// Use structural state in your own logic
if (turbulence > 7000 && energy > 6000) {
    // High turbulence + high energy — adjust behavior
}
```

This is the DeFi composability primitive. Any protocol can incorporate SupraScope's structural model into its own logic — risk parameters, liquidity provision, rebalancing triggers — without trusting a centralized signal.

---

## Deployment Plan

### Phase 1 — Supra Testnet
- Deploy with BTC asset only
- Authorized submitter = development wallet
- History capacity = 2880 epochs (12 hours)
- Validate state update flow end-to-end

### Phase 2 — Supra Mainnet
- Deploy with BTC and ETH
- History capacity = 11520 epochs (2 days)
- Multi-sig admin
- Topology event log live

### Phase 3 — Open Registry
- Asset registration permissioned then open
- Community-submitted assets
- Full history capacity

---

## Storage Cost Estimate

Per asset per epoch (15s):
- Primary state: ~400 bytes
- Memory layers: ~300 bytes
- History entry: ~60 bytes (compressed)
- Total per epoch: ~760 bytes

Per asset per day (5760 epochs):
- ~4.4 MB/day onchain storage

Supra's storage model and gas costs will determine final implementation decisions around history depth and compression strategy.

---

*The patterns reveal themselves.*
