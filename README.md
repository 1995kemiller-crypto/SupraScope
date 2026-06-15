# SupraScope

> A telescope pointed at the hidden geometry of markets.

**supra-scope.com**

**Live now —** BTC and ETH render as real-time geometric objects at supra-scope.com, each driven by a physics engine that logs structural state on a fixed epoch cadence (~15s), 24/7. Months of that history are accumulating in a store nobody else has.

---

## What it is

SupraScope is a perceptual instrument. It transforms the entangled data reality of any asset into a living geometric object — observable, replayable, and structurally honest.

Not a chart. Not a signal. Not a prediction engine.

Six data streams — price, microstructure, derivatives, on-chain, social, and macro — are stacked in the same 4D space and processed through a physics model with memory, gravity, and inertia. The result is a continuously morphing object whose shape encodes the structural character of the asset at this moment in time.

The patterns reveal themselves. SupraScope makes them visible.

---

## Core principles

**One object per asset.** Every asset has its own body, its own characteristic motion, its own baseline geometry. Objects are only comparable to themselves across time.

**Signed spectrum.** The data scale runs −10 to +10. Zero is the midpoint, not the floor. Silence, withdrawal, and absence are first-class data with their own geometric expression.

**Three layers of memory.** Short memory is reactive. Mid memory is contextual — the current regime. Long memory is constitutional — the asset's fundamental behavioral character. The three layers compete and cooperate, never averaging.

**Nominal gravity.** The object learns its own center through simulation. A gravitational field pulls it back toward what it has learned is normal. Deviation is not pathology — it is information density.

**Emergent pattern.** Patterns are not designed in. They crystallize from the relationship between layers. The observer's nervous system does the rest. Pre-linguistic pattern recognition. The shape is legible before it is articulable.

**Agent readable.** The API surface exposes structural narrative, not geometry. An agent receives memory layer divergence, deviation fingerprints, and topology events — and reasons about it with its own intelligence. A new primitive for automated cognition.

---

## The six physics quantities

Each quantity runs on the signed spectrum. They interact through the physics rule set — not through shared raw data.

| Quantity    | What it encodes                         | Geometric effect                         |
| ----------- | --------------------------------------- | ---------------------------------------- |
| Inertia     | Resistance to regime change             | Object moves slowly, resists deformation |
| Tension     | Opposing forces pulling structure apart | Object stretches, edges elongate         |
| Compression | Structure collapsing toward a point     | Vertices pull inward, volume shrinks     |
| Expansion   | Energy entering the system              | Object inflates, rotation accelerates    |
| Turbulence  | Order breakdown, coherence loss         | Surface roughens, connectors fractal     |
| Memory      | Past states shaping current geometry    | Ghost topology echoes prior form         |

---

## Data streams

| Stream         | Sources                                                           | Signed range                           |
| -------------- | ----------------------------------------------------------------- | -------------------------------------- |
| Price          | OHLCV, spot, perp, basis, funding, cross-exchange spread          | −10 withdrawal → +10 active conviction |
| Microstructure | Order book depth, trade flow, bid-ask delta, liquidations         | −10 vacuum → +10 aggressive flow       |
| Derivatives    | OI velocity, vol surface, skew, put/call, gamma exposure          | −10 unwind → +10 conviction build      |
| On-chain       | Wallet flows, exchange inflows, whale moves, new addresses        | −10 outflow → +10 accumulation         |
| Social         | Mention velocity, sentiment, narrative bifurcation, consensus age | −10 silence → +10 viral attention      |
| Macro          | DXY, rates, dominance, event calendar, cross-asset correlation    | −10 risk-off → +10 risk-on             |

---

## Topology events

Rare, unmistakable moments when cohesion between layers breaks.

**Rupture** — two or more streams violently decohere. Connector lines snap outward. Skin surface tears. Liquidation cascade, flash event, order book vacuum.

**Fold** — one axis inverts sign. The object folds through itself. A layer that was positive goes deeply negative while others hold. Regime reversal forming.

**Collapse** — all streams simultaneously compress toward center. The object shrinks to near its core. Connectors dim. Eerie silence across all layers. Liquidity void, pre-event stillness.

**Bloom** — sudden outward expansion across all layers simultaneously. Every stream lights up together. Price discovery, new participant inflow, narrative ignition.

Each topology event leaves a structural fingerprint in long memory. The object has felt this before.

---

## The object as agent primitive

Every agent consuming market data today does so through flat numerical streams. Price, volume, RSI. It is like describing a sculpture over the phone one measurement at a time.

SupraScope gives agents something categorically different — a structured geometric state with signed dimensions, memory layers, nominal reference, and deviation vectors.

Instead of:
> *Price down 3.2%, volume up 40%, RSI 34*

An agent receives:
> *Short memory: high negative turbulence. Mid memory: nominal gravity weakening. Long memory: recognizes this deviation fingerprint. Social layer: −6 withdrawal. Structural layer: positive tension rising. Object distance from nominal: significant and accelerating.*

That is a qualitative structural description of a moment. An agent can reason about that. It has context, dimensionality, memory, and relationship. It is closer to how an experienced human thinks about a market than anything currently possible to encode programmatically.

This is the claim the project takes most seriously, so it is being tested directly: the structural-narrative transform runs against a live model, blind-judged head to head with the raw numeric feed, to measure whether the framing actually produces better agent reasoning.

---

## Rewind

The full state history is retained — every epoch, every asset, since the engine went live. The object can be rewound to any past epoch and replayed, so you feel the structural character of a historical moment rather than read about it. An observer who has replayed significant moments across multiple assets builds a perceptual vocabulary that no dashboard can replicate. That accumulating history is the part of the system that cannot be back-filled or copied.

---

## Architecture

### Running today

```
Market data ingestion  (exchange feeds · on-chain · social)
        ↓
Physics engine — Python, one process per asset  (engine.py · engine_eth.py, on Railway)
   six signed quantities · regime classification · topology detection · nominal gravity
        ↓
State store — Supabase / Postgres
   live state + full epoch history · ~15s cadence · row-level security · indexed
        ↓
┌──────────────────────────────────────────────────────┐
│  Web renderer          │  Realtime feed   │  Narrative   │
│  Three.js · 4D→3D       │  Supabase WS     │  transform    │
│  live morphing object  │  per-epoch push  │  narrate()    │
└──────────────────────────────────────────────────────┘
```

The renderer pulls live state over a Supabase realtime channel and deforms the object per epoch; the same state rows feed the structural-narrative transform.

### Target architecture

The current stack proves the instrument. The destination moves the physics and the state on-chain:

```
Supra oracle feeds  →  Rust physics engine  →  Move ObservatoryState (onchain)  →  renderer · API · agents
```

**Why Supra.** Native oracle infrastructure means no bridge latency. Sub-second finality means the object feels alive. The Move VM's resource model means `ObservatoryState` becomes a first-class onchain primitive — readable and composable by any other contract.

**Why offchain compute.** The ODE solver, Hawkes process estimator, and social ingestion pipeline run offchain. The chain holds the result — verified, signed, permanent. The physics decides the epoch length, not the block time.

---

## Current state

### Live

- BTC and ETH objects rendering in real time at supra-scope.com
- Python physics engine computing all six quantities + regime + topology, running 24/7 per asset
- Live market-data ingestion, structural state logged every epoch
- Per-asset nominal baseline (BTC, ETH)
- Full epoch history accumulating in Postgres — replayable
- Hardened backend — row-level security, indexed latest-epoch lookups, scoped realtime publication
- Theory document at supra-scope.com/theory.html

### In progress

- Structural-narrative transform (`narrate()`) — prototyped and being blind-tested against a live model; endpoint not yet deployed
- Spec docs — API.md, CONTRACT.md, LAYERS.md, PHYSICS.md

### Roadmap

- `/v1/narrative` agent API — serve the transform over live history
- Additional assets (SOL next)
- Cross-asset comparison view
- Physics engine ported to Rust
- `ObservatoryState` anchored on-chain via Supra Move
- Ingestion migrated to Supra native oracle feeds

---

## Built on

### Running today

- **Python** — physics engine
- **Supabase / Postgres** — state store, realtime, full history
- **Railway** — engine hosting
- **Netlify** — static hosting
- **Three.js** — WebGL rendering, 4D→3D projection

### Target stack

- **Supra** — native oracle layer, HyperNova BFT consensus, Move VM (`ObservatoryState` on-chain)
- **Rust** — physics engine port
- **FastAPI** — agent API

---

## Contact

**X** — reach out via X to discuss integration, research access, or collaboration.

**supra-scope.com**

---

*The patterns reveal themselves.*
