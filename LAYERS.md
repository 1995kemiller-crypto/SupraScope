# SupraScope — Layer Interaction, Algorithm, Time & Scale

> Version 0.1 · The bridge between theory and engine

---

## The Core Principle

No layer talks directly to another layer at the data level.

Each stream normalizes its own inputs independently. Each physics quantity draws from multiple streams but through its own weighted equation. The interaction between layers happens at the **physics rule set level** — not at the data level. This is what makes the system coherent rather than noisy.

The object emerges from the interactions. It is not assembled from them.

---

## Layer Architecture

```
RAW DATA LAYER
─────────────────────────────────────────────────────
Price feeds → normalize → price_signal[-10, +10]
Order book  → normalize → micro_signal[-10, +10]
Options     → normalize → deriv_signal[-10, +10]
On-chain    → normalize → chain_signal[-10, +10]
Social      → normalize → social_signal[-10, +10]
Macro       → normalize → macro_signal[-10, +10]

         ↓ (no cross-stream contact here)

PHYSICS QUANTITY LAYER
─────────────────────────────────────────────────────
inertia(price, chain, social)      → I[-10, +10]
tension(price, micro, social)      → T[-10, +10]
compression(micro, deriv, chain)   → C[-10, +10]
expansion(price, chain, social)    → E[-10, +10]
turbulence(micro, social, deriv)   → τ[-10, +10]
memory(all layers via fingerprint) → M[-10, +10]

         ↓ (quantities interact here through physics rules)

STATE VECTOR LAYER
─────────────────────────────────────────────────────
S(t) = [I, T, C, E, τ, M]
evolves via ODE each epoch
shaped by nominal gravity
recorded in three memory layers

         ↓

OUTPUT LAYER
─────────────────────────────────────────────────────
→ Onchain contract (Supra)
→ WebSocket delta stream
→ REST snapshot API
→ Agent briefing
→ Renderer vertex stream
```

---

## How Layers Interact — The Algorithm

### Step 1: Parallel Normalization (all streams simultaneously)

Each stream runs its own normalization independently. No stream waits for another. This is parallelizable — all six streams normalize in the same clock cycle.

```rust
// Pseudocode — runs in parallel threads
let price_signal  = normalize(price_feed,   asset_config.price_window);
let micro_signal  = normalize(order_book,   asset_config.micro_window);
let deriv_signal  = normalize(options_feed, asset_config.deriv_window);
let chain_signal  = normalize(onchain_feed, asset_config.chain_window);
let social_signal = normalize(social_feed,  asset_config.social_window);
let macro_signal  = normalize(macro_feed,   asset_config.macro_window);
```

**Silence detection runs here.** If a stream has been quiet for `silence_threshold` epochs, its signal decays toward its silence baseline rather than holding its last value. Silence is directional, not neutral.

---

### Step 2: Quantity Computation (weighted combination)

Each quantity draws from its designated streams using fixed weights. The weights are asset-specific and defined in configuration — not learned dynamically. This is intentional. Dynamic weight learning would introduce instability and make the system uninterpretable.

```rust
let I = inertia_weights.price    * price_signal
      + inertia_weights.chain    * chain_signal
      + inertia_weights.social   * social_signal;

let T = tension_weights.price    * price_signal
      + tension_weights.micro    * micro_signal
      + tension_weights.social   * social_signal;

let C = compression_weights.micro  * micro_signal
      + compression_weights.deriv  * deriv_signal
      + compression_weights.chain  * chain_signal;

let E = expansion_weights.price    * price_signal
      + expansion_weights.chain    * chain_signal
      + expansion_weights.social   * social_signal;

let τ = turbulence_weights.micro   * micro_signal
      + turbulence_weights.social  * social_signal
      + turbulence_weights.deriv   * deriv_signal
      + hawkes_branching_ratio(price_feed, micro_feed);

let M = memory_match(current_fingerprint, short_memory)  * 0.2
      + memory_match(current_fingerprint, mid_memory)    * 0.4
      + memory_match(current_fingerprint, long_memory)   * 0.4
      - nominal_distance(S_current, S_nominal)           * 0.1;
```

---

### Step 3: State Vector Evolution (ODE step)

The six quantities are not used raw. They feed into a damped dynamical system that gives the object physical continuity — inertia, resistance to sudden change, smooth transitions.

```
S(t+1) = S(t) 
       + dt * forces(S(t))           // raw quantity targets
       - dt * damping * S(t)         // resistance to extremes  
       + dt * gravity(S(t), N)       // pull toward nominal
       + dt * noise(τ)               // turbulence-scaled stochastic term
```

Where `dt = epoch_length / physics_timescale`

**This is why the object feels alive rather than twitchy.** The ODE smooths the transitions. A sudden spike in micro_signal doesn't immediately spike the state — it creates a force that the damped system responds to gradually. The object has mass.

**Turbulence modulates everything:**
```
effective_damping = base_damping * (1 + I * 0.05)
effective_gravity = base_gravity * (1 - |τ| / 10)
effective_noise   = base_noise   * (1 + |τ| / 5)
```

High turbulence → reduces gravity (can't find home), increases noise (chaotic surface), barely affects damping (the chaos is real, not just resistance).

---

### Step 4: Memory Layer Update

After each state vector update, all three memory layers receive the new fingerprint.

```
fingerprint = compress(round(S, 1))   // 6 values, 1 decimal place

short_memory.append(fingerprint, weight=1.0,   decay=exponential(half_life=16))
mid_memory.append(fingerprint,   weight=1.0,   decay=exponential(half_life=288))
long_memory.append(fingerprint,  weight=1.0,   decay=power_law(significant_events_decay_slower))
```

**Long memory is not just old short memory.** It uses a different decay function. Routine states decay normally. States that preceded topology events decay slower — they become more persistent because they are more meaningful. The long memory is weighted by consequence, not just recency.

---

### Step 5: Nominal State Update

The nominal state updates slowly — it is not recomputed every epoch. It recalculates every 96 epochs (24 minutes) using the most recent low-energy states.

```
if epoch % 96 == 0:
    low_energy_states = history.filter(energy < NOMINAL_THRESHOLD)
    N = weighted_centroid(low_energy_states, weights=recency_decay)
    nominal_confidence = min(1.0, history_length / CONFIDENCE_EPOCHS)
```

The nominal state has **confidence** that grows over time. A new asset has low confidence — its nominal gravity is weak because the system hasn't seen enough of its behavioral range. An asset with a year of history has high confidence — the nominal state is well established and gravity is strong.

---

### Step 6: Topology Event Detection

After state update, run topology checks:

```
rupture_check(S, dS, stream_correlations)   → fire if conditions met
fold_check(S, dS, I)                         → fire if conditions met
collapse_check(S, dS, E)                     → fire if conditions met
bloom_check(S, dS, all_quantities)           → fire if conditions met
```

Topology events are **rare and unmistakable by design.** Thresholds are set conservatively. It is better to miss an event than to fire false positives — false positives train observers to ignore the signal.

---

## Time

Time is the dimension that makes everything else meaningful.

### Epoch Time (15 seconds — BTC)
The fundamental tick. Every 15 seconds:
- All streams sample their latest data
- All quantities recompute
- State vector takes one ODE step
- Memory layers update
- Nominal state rechecks (every 96 epochs)
- Output pushed to contract and stream

**Why 15 seconds?** It matches the natural timescale of microstructure resolution for BTC. Fast enough to catch order book dynamics. Slow enough that social and macro inputs (which update less frequently) don't create stale data problems. The physics engine interpolates slower inputs between their update frequencies.

### Physics Time (continuous within epoch)
The ODE solver runs in continuous time within each epoch. The 15-second epoch is the sampling rate. The physics operates on a finer internal timescale to ensure numerical stability.

```
physics_substeps = 8    // 8 internal steps per epoch
dt_internal = epoch_length / physics_substeps  // 1.875 seconds
```

### Memory Time (three timescales)

| Layer | Timescale | Half-life | Purpose |
|---|---|---|---|
| Short | 0–16 min | 4 min | Immediate texture, reactive |
| Mid | 16 min–5 hr | 72 min | Current regime character |
| Long | 5 hr–permanent | Power law | Constitutional character |

These three timescales create **temporal depth**. The object at any moment is simultaneously shaped by what happened seconds ago, hours ago, and months ago. Each memory layer pulls with different force. When they agree the object is settled. When they disagree the object is in transition.

### Historical Time (rewind)
Every epoch is stored onchain. The object can be rewound to any moment in its history and replayed forward. This creates a completely new relationship with market history — not chart replay, but structural re-experience.

An observer can inhabit the structural character of March 2020, feel the collapse and bloom topology events in sequence, and carry that embodied memory into reading the present object.

### Asset Time (lifecycle)
Each asset has its own temporal lifecycle in the system:

```
Birth       → nominal_confidence = 0.0, weak gravity, high uncertainty
Adolescence → nominal_confidence 0.0–0.5, growing gravity, patterns emerging  
Maturity    → nominal_confidence 0.5–1.0, strong gravity, clear character
Established → nominal_confidence = 1.0, full gravity, rich historical rhyme library
```

BTC is established on day one — we import historical data to pre-train its nominal state before the live engine starts. A new DeFi protocol starts at birth and matures in real time.

---

## Scale

### Horizontal Scale — Multiple Assets

Each asset runs as an independent physics engine instance. Adding ETH is spinning up a new instance with ETH configuration. Assets do not share compute.

```
engine_btc  = PhysicsEngine::new(config/BTC.yaml)
engine_eth  = PhysicsEngine::new(config/ETH.yaml)
engine_sol  = PhysicsEngine::new(config/SOL.yaml)
engine_aave = PhysicsEngine::new(config/AAVE.yaml)
```

Each instance:
- Has its own data subscriptions
- Has its own normalization parameters
- Has its own nominal state
- Writes to its own onchain resource
- Costs approximately the same compute regardless of other assets

**Adding 10 assets costs 10x the compute of 1 asset.** Linear scaling. No cross-asset coupling in the physics layer. Cross-asset relationships emerge at the combined object layer — a separate, optional compute layer that reads two or more asset states and computes their entanglement.

### Vertical Scale — Resolution

Each asset can run at different epoch resolutions:

| Resolution | Epoch | Use case |
|---|---|---|
| Micro | 1 second | HFT-adjacent agents, deep microstructure |
| Standard | 15 seconds | Default — BTC, ETH major assets |
| Macro | 5 minutes | Lower liquidity assets, DeFi protocols |
| Slow | 1 hour | Macro assets, cross-market structural views |

Higher resolution = more compute, more onchain writes, more data cost. Lower resolution = cheaper, suitable for assets with slower-moving data.

### Data Scale — Stream Depth

Each stream can operate at different depths:

| Depth | Description | Cost |
|---|---|---|
| Minimal | Price only, no microstructure | Free |
| Standard | Price + microstructure + derivatives | ~$35/month |
| Full | All six streams including social | ~$100/month |
| Research | Full + alternative data, satellite, etc. | Custom |

The physics engine degrades gracefully when streams are missing — absent streams contribute their silence baseline rather than crashing the computation.

---

## Why It's Faster — The Speed Argument

This is the fundamental case for SupraScope as agent infrastructure.

### Problem: 2D data processing is architecturally slow

A conventional agent consuming market data must:

```
1. Receive raw tick data (price, volume, order book)
2. Store in rolling buffer
3. Compute features (returns, ratios, indicators)
4. Normalize features
5. Detect regimes
6. Assess context (what kind of moment is this?)
7. Build reasoning prompt
8. Reason
9. Decide
```

Steps 1–7 are preprocessing. The agent hasn't started thinking yet.

This pipeline:
- Runs sequentially — each step waits for the previous
- Is implemented differently by every agent developer — inconsistent, error-prone
- Produces a flat feature vector that loses spatial and temporal relationships
- Has no memory of structural character — only statistical history
- Cannot detect topology events without implementing its own detection logic

**The agent is doing SupraScope's job badly, slowly, every time it runs.**

### Solution: 4D object as pre-integrated structural state

A SupraScope-connected agent receives:

```
1. Structural briefing (pre-computed, pre-integrated)
2. Reason
3. Decide
```

Steps 1–7 above are replaced by a single API call that returns a complete structural description of this moment.

```
Conventional pipeline latency: 200–800ms (preprocessing + LLM)
SupraScope pipeline latency:   50–150ms  (API call + LLM)

Reasoning quality: conventional agent reasons about numbers
                   SupraScope agent reasons about structure
```

**It's not just faster. It's reasoning about a fundamentally richer representation.**

### The Dimensional Argument

A 2D data point carries 1 relationship (value over time).

A 4D object state carries:
- 6 signed quantities (6 dimensions of structural information)
- 3 memory layers × 6 quantities (18 historical reference points)
- Nominal state + deviation vector (direction and magnitude of departure)
- Topology proximity (how close to a structural phase transition)
- Historical rhymes (which past moments this resembles)
- Regime label (qualitative character of this moment)

**One API call returns what would take hundreds of individual data calls to approximate — and still wouldn't capture the spatial and temporal relationships.**

This is the 1s and 0s vs quantum data argument made precise:

```
2D agent input:  [price_t1, vol_t1, price_t2, vol_t2, ... price_tN, vol_tN]
                 Sequential. Flat. Agent must integrate.

4D object input: {
                   state: {I, T, C, E, τ, M},
                   memory: {short, mid, long},
                   nominal: {N, confidence, gravity},
                   deviation: {magnitude, direction, velocity},
                   topology: {proximity, last_event},
                   rhymes: [{similarity, what_followed}]
                 }
                 Simultaneous. Structured. Pre-integrated.
                 Agent reasons immediately.
```

### The Hand Signal Analogy

An experienced trader's hand signal to a clerk contains:
- Current structural assessment
- Urgency
- Direction
- Confidence level
- Historical context ("I've seen this before")

All in one gesture. Transmitted in milliseconds. Received and understood without decomposition.

SupraScope's agent briefing is the programmatic equivalent. It carries the full structural assessment of a market moment in a single structured object. The agent receives it the way the clerk receives the hand signal — complete, pre-integrated, immediately actionable.

**No decomposition required. No preprocessing pipeline. Just reasoning.**

---

## Fluidity

The object's fluidity — the smooth continuous motion rather than discrete jumps — comes from three design decisions:

**1. ODE with damping.** The state vector doesn't snap to new values. It moves toward them through a damped dynamical system. Sudden data spikes create forces, not teleportations.

**2. Parallel stream normalization.** Because streams normalize independently on rolling windows, no single data point causes a discontinuous jump. The window smooths incoming data before it touches the physics.

**3. Memory gravity.** The nominal state pulls the object back toward familiar territory. Combined with damping this creates a system that departs smoothly from nominal and returns smoothly — no hard resets.

The result is an object that moves the way a physical system moves. It has weight. It has momentum. It resists sudden change. When it does change dramatically — topology events — the departure is meaningful precisely because it overcame the system's natural resistance.

**Fluidity is not aesthetic. It is informational.** The speed and smoothness of a state transition carries as much information as the transition itself.

---

## Summary

| Dimension | Design choice | Why |
|---|---|---|
| Layer interaction | Via physics rules, not direct data sharing | Prevents noise cascade |
| Normalization | Per-stream, rolling z-score with silence | Comparable signed values |
| Time | 15s epoch, 8 substeps, 3 memory timescales | Matches BTC microstructure |
| Scale | Linear horizontal, configurable vertical | Predictable cost growth |
| Speed | Pre-integrated structural state | Removes agent preprocessing |
| Fluidity | ODE + damping + memory gravity | Informational continuity |

---

*The patterns reveal themselves.*
