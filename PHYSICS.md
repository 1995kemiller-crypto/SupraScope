# SupraScope — Physics Engine Specification

> Version 0.1 · BTC initial implementation

---

## Overview

The physics engine is the core of SupraScope. It ingests normalized data streams, computes six signed quantities, evolves a latent state vector through a physics rule set, and outputs a continuous state that the renderer, API, and onchain contract consume.

The engine runs offchain. Its output is submitted onchain each epoch as a signed transaction. The chain holds the result — verified, permanent, composable.

---

## Signed Spectrum

Every quantity and every input operates on the signed spectrum:

```
−10 ←————————— 0 ————————————→ +10
withdrawal     neutral        active presence
silence        midpoint       conviction
vacuum         baseline       energy
```

Zero is not the floor. Zero is the center. Negative readings are first-class data — they encode withdrawal, silence, vacuum, and anti-presence. A stream that goes dark is not missing — it is speaking.

---

## Epoch

The physics engine updates on a fixed epoch interval. For BTC initial implementation:

```
epoch_length = 15 seconds
```

Each epoch:
1. Ingest latest data from all streams
2. Normalize each input to signed spectrum
3. Compute six physics quantities
4. Evolve state vector via physics rules
5. Update three memory layers
6. Compute nominal gravity vector
7. Submit state to Supra contract
8. Push delta to WebSocket stream

---

## Input Normalization

Every raw input is normalized to the signed spectrum before entering the physics model. Normalization is per-asset, per-timeframe, using a rolling z-score with decay:

```
normalized(x, t) = clamp(z_score(x, window=96_epochs) * scale, -10, 10)
```

Where:
- `window` = 96 epochs (24 minutes at 15s epochs)
- `scale` = 3.0 (maps ±3σ to ±10)
- `clamp` = hard limits at −10 and +10

**Silence detection.** When a stream produces no data for `silence_threshold` epochs, its normalized value decays toward its silence baseline (typically negative) at a configurable rate. Silence is not zero — it is a directional signal.

```
silence_baseline = −4.0   (BTC default — active asset going quiet is meaningful)
silence_decay_rate = 0.15 per epoch
```

---

## The Six Physics Quantities

### 1. Inertia `I ∈ [−10, +10]`

Resistance of the current state to change. High positive inertia means the system is deeply entrenched in its current regime. High negative inertia means the system is actively destabilizing — resistant to settling.

**Inputs:**
- Price autocorrelation over 32 epochs (primary)
- Open interest persistence (secondary)
- Social consensus age — how long the dominant narrative has been stable (tertiary)

**Equation:**
```
I(t) = 0.6 * autocorr(price, 32) 
     + 0.3 * oi_persistence(t) 
     + 0.1 * consensus_age(t)
```

**Geometric effect:** High inertia → object resists deformation, moves slowly, rotation dampened. Negative inertia → object becomes volatile, susceptible to rapid topology change.

---

### 2. Tension `T ∈ [−10, +10]`

Opposing forces pulling the structure apart. Positive tension means active divergence between forces. Negative tension means coordinated withdrawal — forces that were opposing each other have both left. Negative tension is more dangerous than positive.

**Inputs:**
- Bid-ask spread normalized (primary)
- Spot-perp basis magnitude (primary)
- Narrative bifurcation score — bimodality of topic clusters (secondary)
- Cross-exchange price divergence (tertiary)

**Equation:**
```
T(t) = 0.4 * spread_normalized(t)
     + 0.3 * basis_magnitude(t)
     + 0.2 * narrative_bifurcation(t)
     + 0.1 * cross_exchange_div(t)
```

**Geometric effect:** Positive tension → object stretches along tension axis, edges elongate. Negative tension → coordinated contraction, eerie stillness, skin surface smooths unnaturally.

---

### 3. Compression `C ∈ [−10, +10]`

Structure collapsing toward a point. Positive compression means liquidity and participation are thinning. Negative compression means active anti-compression — forced expansion against a contracting background.

**Inputs:**
- Order book depth thinning rate (primary)
- Implied volatility crush — IV dropping sharply (primary)
- Trade size shrinkage (secondary)
- New wallet address rate declining (tertiary)

**Equation:**
```
C(t) = 0.4 * book_depth_delta(t)
     + 0.35 * iv_crush(t)
     + 0.15 * trade_size_delta(t)
     + 0.10 * wallet_rate_delta(t)
```

**Geometric effect:** Positive compression → vertices pull inward, object volume shrinks, core brightens. Negative compression → forced outward pressure, skin stretches against contracting field.

---

### 4. Expansion `E ∈ [−10, +10]`

Energy entering the system. Positive expansion means genuine participation growth. Negative expansion means active capital and attention withdrawal — the market is shrinking.

**Inputs:**
- Volume delta (primary)
- Open interest growth rate (primary)
- New wallet inflow rate (secondary)
- Search trend acceleration (secondary)
- Exchange inflow rate (tertiary)

**Equation:**
```
E(t) = 0.35 * volume_delta(t)
     + 0.30 * oi_growth(t)
     + 0.15 * wallet_inflow(t)
     + 0.12 * search_accel(t)
     + 0.08 * exchange_inflow(t)
```

**Geometric effect:** Positive expansion → object inflates, rotation accelerates, connectors brighten. Negative expansion → field contracts, particles draw inward, connectors dim.

---

### 5. Turbulence `τ ∈ [−10, +10]`

Order breakdown and coherence loss. Positive turbulence means chaotic, decorrelated activity. Negative turbulence is the most unusual state — eerie coherence, streams locking together in unnatural stillness. Both extremes are pre-transition signals.

**Inputs:**
- Flow sign reversal rate — how often buy/sell delta flips (primary)
- Cross-stream decorrelation score (primary)
- Liquidation cascade magnitude (secondary)
- Social sentiment polarity swings (tertiary)

**Equation:**
```
τ(t) = 0.40 * flow_reversal_rate(t)
     + 0.35 * cross_stream_decorr(t)
     + 0.15 * liquidation_heat(t)
     + 0.10 * sentiment_swing(t)
```

**Hawkes branching ratio.** Turbulence also incorporates a Hawkes process self-excitation estimate — measuring whether the current activity is self-reinforcing (branching ratio → 1.0 means critical, approaching cascade).

```
τ_hawkes(t) = (branching_ratio(t) − 0.5) * 20   // maps 0.0–1.0 to −10–+10
τ_final(t)  = 0.7 * τ(t) + 0.3 * τ_hawkes(t)
```

**Geometric effect:** Positive turbulence → surface roughens, high-frequency vertex displacement, connectors fractal. Negative turbulence → eerie coherence, streams lock, unnatural smoothness that precedes rupture.

---

### 6. Memory `M ∈ [−10, +10]`

The influence of past states on current geometry. Positive memory means the system is actively recalling and being shaped by prior configurations. Negative memory means the system is in genuinely novel territory — no strong historical fingerprint matches the current state.

**Inputs:**
- Short memory layer agreement with current state
- Mid memory layer agreement with current state
- Long memory layer agreement with current state
- Nominal state distance (inverse — far from nominal weakens memory pull)

**Equation:**
```
M(t) = 0.2 * short_match(t)
     + 0.4 * mid_match(t)
     + 0.4 * long_match(t)
− 0.1 * nominal_distance(t)
```

**Geometric effect:** High positive memory → object pulled toward familiar configurations, ghost topology visible, nominal gravity strong. Negative memory → object in uncharted territory, nominal gravity weakens, topology unpredictable.

---

## Three Memory Layers

### Short Memory `S`
- **Timescale:** 0–64 epochs (~16 minutes at 15s)
- **Decay:** Exponential, half-life = 16 epochs
- **Role:** Immediate echo. Reactive. Holds the texture of the last few minutes.
- **Weight in gravity:** 0.2

### Mid Memory `M`
- **Timescale:** 64–1152 epochs (~16 minutes to ~5 hours)
- **Decay:** Exponential, half-life = 288 epochs
- **Role:** Current regime context. Is this a trending, ranging, or fragmenting state.
- **Weight in gravity:** 0.4

### Long Memory `L`
- **Timescale:** 1152+ epochs (~5 hours to permanent)
- **Decay:** Power law — significant events decay slowly, routine states decay faster
- **Role:** Constitutional character. BTC's long memory encodes its full behavioral range.
- **Weight in gravity:** 0.4

### State Fingerprinting
Each epoch produces a compact state fingerprint:
```
fingerprint = hash(round(I, 1), round(T, 1), round(C, 1), 
                   round(E, 1), round(τ, 1), round(M, 1))
```

Fingerprints are stored in all three memory layers with their timestamps and subsequent topology events. When a current fingerprint matches a historical one within a similarity threshold, memory activation fires and influences the current state.

---

## Nominal State

The nominal state `N` is the object's learned center of gravity — its characteristic resting form computed from the history of low-energy states.

```
N(t) = weighted_centroid(states where |energy(s)| < threshold, 
                          weights = recency_decay(t - s.timestamp))
```

**Energy** of a state is defined as:
```
energy(s) = sqrt(I² + T² + C² + E² + τ² + M²) / sqrt(6)
```

Low energy states (energy < 3.0) are the states the object "considers normal." The nominal state is their recency-weighted centroid.

### Nominal Gravity Vector
```
G(t) = (N - current_state) * gravity_strength(t)
```

Where:
```
gravity_strength(t) = base_gravity * confidence(N) * (1 - |τ| / 10)
```

- `base_gravity = 0.08` (BTC initial)
- `confidence(N)` grows from 0 → 1 as simulation history accumulates
- High turbulence reduces gravity — the object can't find home during chaos

---

## Physics ODE

The state vector evolves each epoch via a damped system:

```
dS/dt = forces(S, t) − damping * S + gravity(S, N, t) + noise(τ, t)
```

Where:
- `forces` = weighted sum of normalized inputs driving each quantity
- `damping = 0.12` (BTC initial — prevents runaway divergence)
- `gravity` = pull toward nominal state
- `noise` = turbulence-scaled stochastic term (zero-mean, variance ∝ |τ|)

**Inertia modifies damping:**
```
effective_damping = damping * (1 + I * 0.05)
```

High inertia increases damping — the system resists change more strongly.

---

## Topology Event Detection

Topology events are detected by monitoring state vector derivatives and cross-quantity interactions.

### Rupture
```
condition: d(cross_stream_decorr)/dt > rupture_threshold
           AND |τ| > 7.0
           AND two or more quantities change sign within 3 epochs
threshold: 0.4 per epoch (BTC)
```

### Fold
```
condition: any single quantity crosses zero with velocity > fold_velocity
           AND opposing quantity holds sign
           AND |I| < 3.0 (low inertia — system not resisting)
velocity:  2.0 units per epoch (BTC)
```

### Collapse
```
condition: ALL six quantities simultaneously move toward zero
           AND energy(S) declining for 8+ consecutive epochs
           AND |E| < −4.0 (active withdrawal)
```

### Bloom
```
condition: ALL six quantities simultaneously positive and increasing
           AND d(energy)/dt > bloom_threshold for 3+ consecutive epochs
threshold: 0.8 per epoch (BTC)
```

---

## BTC Initial Parameters

```yaml
asset: BTC-USDT
epoch_length: 15s
silence_baseline: -4.0
silence_decay_rate: 0.15
base_gravity: 0.08
damping: 0.12
normalization_window: 96 epochs
normalization_scale: 3.0
hawkes_weight: 0.3
nominal_energy_threshold: 3.0
nominal_confidence_epochs: 2880  # ~12 hours to full confidence
```

---

## Data Sources — BTC Initial

| Stream | Source | Update frequency |
|---|---|---|
| Price / Microstructure | Binance WebSocket BTCUSDT | Per tick |
| Derivatives | Binance perp + Deribit options | Per tick / 1min |
| On-chain | Glassnode API | 10min |
| Social | Lunarcrush API / CT scraper | 5min |
| Macro | CoinGecko + FRED API | 15min |
| Canonical price | Supra oracle | Per block |

---

## Adding New Assets

The physics engine is asset-agnostic. Adding ETH or any other asset requires:

1. Asset-specific normalization parameters (silence baseline, window size)
2. Asset-specific gravity strength (illiquid assets need weaker gravity)
3. Asset-specific topology thresholds (small caps rupture at lower thresholds)
4. Sufficient simulation history to establish nominal state

Configuration lives in `assets/{symbol}.yaml`. The engine loads it at initialization. No code changes required.

---

*The patterns reveal themselves.*
