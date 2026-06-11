# SupraScope — API Schema Specification

> Version 0.1 · BTC initial implementation

---

## Overview

The SupraScope API exposes the onchain state as a structured perception interface. It does not serve geometry — it serves structural narrative. Every consumer reads the same canonical state verified against the Supra contract.

Three consumer types:

- **Renderer** — needs full state at high frequency for visual update
- **Agent** — needs qualitative structural description for reasoning
- **Researcher** — needs historical access, range queries, pattern search

---

## Base URL

```
https://api.supra-scope.com/v1
```

All responses are JSON. All timestamps are Unix milliseconds. All quantity values are signed floats in range [−10.0, +10.0].

---

## Authentication

Public endpoints (read-only state) require no authentication for launch phase.

Agent endpoints (structural briefing) require an API key header:
```
X-SupraScope-Key: your_api_key
```

---

## Endpoints

---

### GET `/object/{asset}/state`

Current full state of an asset object.

**Parameters:**
- `asset` — asset identifier e.g. `BTC`, `ETH`

**Response:**
```json
{
  "asset": "BTC",
  "epoch": 48291,
  "timestamp": 1718023847000,
  "onchain_tx": "0xabc...def",
  "state": {
    "inertia":     2.4,
    "tension":    -1.8,
    "compression": 0.3,
    "expansion":   4.1,
    "turbulence":  1.2,
    "memory":      3.7
  },
  "memory_layers": {
    "short": {
      "inertia":     2.1,
      "tension":    -2.0,
      "compression": 0.1,
      "expansion":   3.8,
      "turbulence":  1.4,
      "memory":      2.9
    },
    "mid": {
      "inertia":     1.8,
      "tension":    -0.9,
      "compression": 0.8,
      "expansion":   2.2,
      "turbulence":  0.6,
      "memory":      3.1
    },
    "long": {
      "inertia":     1.2,
      "tension":     0.4,
      "compression": 1.1,
      "expansion":   1.4,
      "turbulence":  0.2,
      "memory":      3.4
    }
  },
  "nominal": {
    "inertia":     1.1,
    "tension":     0.2,
    "compression": 0.9,
    "expansion":   1.2,
    "turbulence":  0.1,
    "memory":      2.8,
    "confidence":  0.94
  },
  "deviation": {
    "magnitude":   3.2,
    "direction":   [0.4, -0.6, -0.1, 0.8, 0.3, 0.2],
    "velocity":    0.18,
    "accelerating": true
  },
  "energy": 2.8,
  "topology": {
    "active_event": null,
    "last_event": {
      "type": "fold",
      "epoch": 48105,
      "timestamp": 1718021210000,
      "magnitude": 4.2
    }
  },
  "regime": "expanding_coherent"
}
```

---

### GET `/object/{asset}/state/{epoch}`

State at a specific historical epoch. Full rewind access.

**Response:** Same schema as current state.

---

### GET `/object/{asset}/state/range`

State across a time range. For research and replay.

**Query parameters:**
- `from` — Unix ms timestamp
- `to` — Unix ms timestamp
- `resolution` — `epoch` | `minute` | `hour` | `day`

**Response:**
```json
{
  "asset": "BTC",
  "from": 1718000000000,
  "to": 1718023847000,
  "resolution": "minute",
  "count": 398,
  "states": [
    { "epoch": 47894, "timestamp": 1718000015000, "state": { ... } },
    ...
  ]
}
```

---

### GET `/object/{asset}/layers/{layer}`

Single layer in isolation. For consumers that only need one dimension.

**Parameters:**
- `layer` — `physics` | `memory` | `nominal` | `deviation` | `topology` | `social` | `streams`

**Example — `/object/BTC/layers/streams`:**
```json
{
  "asset": "BTC",
  "timestamp": 1718023847000,
  "streams": {
    "price": {
      "value": 3.8,
      "delta": 0.4,
      "silence": false
    },
    "microstructure": {
      "value": -1.2,
      "delta": -0.8,
      "silence": false
    },
    "derivatives": {
      "value": 2.1,
      "delta": 0.2,
      "silence": false
    },
    "onchain": {
      "value": 1.4,
      "delta": 0.1,
      "silence": false
    },
    "social": {
      "value": -3.8,
      "delta": -1.2,
      "silence": true,
      "silence_epochs": 8
    },
    "macro": {
      "value": 0.6,
      "delta": 0.0,
      "silence": false
    }
  }
}
```

---

### GET `/object/{asset}/agent`

Structural narrative briefing for agent consumption. The most important endpoint. Returns a qualitative description of the current moment designed for LLM reasoning.

**Response:**
```json
{
  "asset": "BTC",
  "timestamp": 1718023847000,
  "epoch": 48291,
  "moment_character": "expanding_with_social_withdrawal",
  "energy_level": "moderate",
  "signed_summary": {
    "dominant_positive": ["expansion", "memory"],
    "dominant_negative": ["tension", "social_stream"],
    "near_zero": ["compression", "turbulence"]
  },
  "memory_assessment": {
    "short_mid_agreement": true,
    "mid_long_agreement": false,
    "all_aligned": false,
    "divergence_note": "mid and long memory disagree on tension — mid reads negative, long reads neutral. Regime may be newer than it feels."
  },
  "nominal_relationship": {
    "distance": "moderate",
    "direction": "expanding away from nominal",
    "velocity": "gradual",
    "gravity_strength": "normal"
  },
  "silence_signals": [
    {
      "stream": "social",
      "epochs_silent": 8,
      "significance": "social layer withdrawing while price expanding — crowd absent from this move"
    }
  ],
  "topology_proximity": {
    "nearest_event_type": "bloom",
    "proximity": "low",
    "conditions_present": 1,
    "conditions_required": 3
  },
  "historical_rhymes": [
    {
      "epoch": 31204,
      "timestamp": 1715882400000,
      "similarity": 0.87,
      "what_followed": "object continued expanding for 18 epochs then compressed sharply",
      "note": "social was also silent during that configuration"
    }
  ],
  "structural_note": "Expansion with memory agreement but social withdrawal. Structure moving without crowd participation. Mid and long memory diverge on tension — this regime may be less established than energy level suggests."
}
```

---

### GET `/object/combined/{assetA}/{assetB}`

Combined two-asset object. Reveals structural dependencies and entanglement.

**Response:**
```json
{
  "assets": ["BTC", "ETH"],
  "timestamp": 1718023847000,
  "entanglement": {
    "coherence_score": 0.72,
    "dominant_relationship": "correlated_expansion",
    "divergence_axes": ["tension", "social"],
    "interference_pattern": "constructive"
  },
  "btc_state": { ... },
  "eth_state": { ... },
  "delta": {
    "inertia":     0.8,
    "tension":    -2.4,
    "compression": 0.2,
    "expansion":  -0.6,
    "turbulence":  1.8,
    "memory":      0.4
  }
}
```

---

### GET `/stream/{asset}`

WebSocket endpoint. Continuous delta stream. Connect once, receive every epoch update.

**Connection:**
```
wss://api.supra-scope.com/v1/stream/BTC
```

**Message format (every epoch):**
```json
{
  "type": "epoch_update",
  "asset": "BTC",
  "epoch": 48292,
  "timestamp": 1718023862000,
  "delta": {
    "inertia":     0.1,
    "tension":    -0.3,
    "compression": 0.0,
    "expansion":   0.2,
    "turbulence":  0.4,
    "memory":      0.1
  },
  "full_state": { ... },
  "topology_event": null
}
```

**Topology event message:**
```json
{
  "type": "topology_event",
  "asset": "BTC",
  "epoch": 48301,
  "timestamp": 1718023997000,
  "event": {
    "type": "rupture",
    "magnitude": 6.8,
    "streams_involved": ["microstructure", "derivatives"],
    "description": "Microstructure and derivatives violently decoherent. Liquidation cascade signature."
  }
}
```

---

### GET `/stream/topology`

WebSocket. Topology events only, across all assets. Lightweight feed for agents monitoring for significant state changes.

```
wss://api.supra-scope.com/v1/stream/topology
```

---

### GET `/object/{asset}/topology`

Historical topology events for an asset.

**Query parameters:**
- `from` — Unix ms
- `to` — Unix ms
- `type` — filter by event type: `rupture` | `fold` | `collapse` | `bloom`
- `min_magnitude` — float, minimum event magnitude

**Response:**
```json
{
  "asset": "BTC",
  "count": 12,
  "events": [
    {
      "type": "collapse",
      "epoch": 41204,
      "timestamp": 1716882400000,
      "magnitude": 7.2,
      "duration_epochs": 14,
      "preceded_by": "eerie_coherence_negative_turbulence",
      "followed_by": "bloom"
    }
  ]
}
```

---

### GET `/object/{asset}/patterns`

Search for historical configurations similar to the current state. The memory system made queryable.

**Query parameters:**
- `similarity` — float 0.0–1.0, minimum match threshold (default 0.8)
- `limit` — max results (default 10)

**Response:**
```json
{
  "asset": "BTC",
  "current_fingerprint": "a3f8c2...",
  "matches": [
    {
      "epoch": 31204,
      "timestamp": 1715882400000,
      "similarity": 0.87,
      "state": { ... },
      "subsequent_topology": {
        "epochs_until_event": 18,
        "event_type": "compression",
        "event_magnitude": 4.1
      }
    }
  ]
}
```

---

### GET `/assets`

List all available assets with their object status.

**Response:**
```json
{
  "assets": [
    {
      "symbol": "BTC",
      "status": "live",
      "nominal_confidence": 0.94,
      "epochs_history": 48291,
      "last_topology_event": "fold",
      "current_energy": 2.8
    }
  ]
}
```

---

## Regime Labels

The `regime` field in the state response uses these named labels:

| Label | Description |
|---|---|
| `nominal` | All layers near nominal state, low energy |
| `expanding_coherent` | Expansion with layer agreement |
| `expanding_fragmented` | Expansion but layers diverging |
| `compressing` | Compression dominant |
| `high_tension` | Tension dominant, opposing forces active |
| `turbulent` | Turbulence dominant, coherence low |
| `eerie_coherent` | Negative turbulence — unnatural stillness |
| `memory_active` | Strong historical fingerprint match |
| `novel_territory` | Low memory — no strong historical match |
| `pre_topology` | Multiple topology conditions approaching threshold |

---

## Onchain Verification

Every state served by the API is verifiable against the Supra contract. The `onchain_tx` field in every response contains the transaction hash that wrote this state. Any consumer can independently verify the state by reading the contract directly.

```
Contract: ObservatoryState on Supra mainnet
Read: get_state(asset_id, epoch)
```

---

## Rate Limits

| Endpoint | Limit |
|---|---|
| REST state endpoints | 120 req/min |
| Agent briefing | 60 req/min |
| Historical range | 20 req/min |
| WebSocket streams | 10 concurrent connections |

---

*The patterns reveal themselves.*
