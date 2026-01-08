# NASDAQ Options Ground Truth & Ingestion Primitive

This repository captures the **Data Oracle and Ingestion Layer** specification for a high-frequency **NASDAQ options** trading environment. It defines the **Data Ground Truth** contract required by downstream layers (scoring engine, trader, HUD) and the validation rules that prevent repainting.

## Codex Prompt: NASDAQ Options Ground Truth & Ingestion Primitive

**Infrastructure Target:** High-performance **NASDAQ (OPRA)** feed integration via low-latency providers (e.g., Polygon.io, FMP).
**Execution Constraint:** **Sub-1ms processing latency** for "A+ Setup" identification.

### 1. Ultra-Low Latency Ingestion (The Tickerplant)
- **Tick-to-Bar Aggregator:** Ingest raw NASDAQ options tick data and aggregate into strictly confirmed candle bars for the **5m, 15m, 1h, and 2h timeframes**.
- **Strict Confirmation Protocol:** To eliminate "repainting" or "ghost signals," no institutional footprint shall be passed to the scoring engine until the **close of the third candle** in the sequence is authenticated.
- **Options-Specific Metadata:** Attach real-time **Average True Range (ATR)** data to each bar to facilitate immediate volatility-based threshold filtering.

### 2. Hard-Coded Institutional Footprint Logic
The primitive must internally validate the following "locked" operational definitions before any scoring occurs:
- **Wick-Based 3-Candle Sequence:**
  - **Bullish:** `High_{n-2} < Low_n`.
  - **Bearish:** `Low_{n-2} > High_n`.
- **Confirmed Displacement Filter:** Flag only sequences where the middle candle exhibits a **body-to-wick ratio > 70%**, signifying forceful institutional pressure.
- **Liquidity Sweep Detection:** Identify candles that "grab" liquidity by taking out recent swing highs or lows before the displacement.

### 3. State-Machine Tracking (Lifecycle Layer)
- **FVG State Management:** Monitor the **Lifecycle** (New → Unmitigated → Touched → Filled) of every identified NASDAQ footprint.
- **Mitigation Logic:** Immediately flag a zone as "Filled" if the current price action completely overlaps the established gap, removing it from the active **ATC Radar**.

### 4. Strategic Multi-Pillar IRR Filtering
Align the ingestion output with the redefined **Flash Trade IRR** framework:
- **Element A:** Calculate projected **Financial Return (NPV)** based on the spread between the identified Order Block and the next swing point.
- **Element B (Efficiency):** Prioritize setups targeting **liquidity pools** where retail stop-loss orders cluster to increase order fulfillment probability.
- **Element C (Catalytic):** Score the potential for the footprint to trigger a **Market Structure Shift (MSS)** on higher timeframes.

### 5. Standardized HUD Data Stream
Output validated setups in an aligned, monospace format for the Glanceable HUD:

```
Ticker | Strike | Expiry | TF | Zone | Status | Distance
```

---

## Data Ground Truth Contract (Interface-Level)
Downstream layers consume **only** validated footprints that satisfy the confirmation protocol and locked definitions. Every emitted record must include:

- **Identity:** `ticker`, `underlier`, `strike`, `expiry`, `option_type` (call/put).
- **Time:** `tf`, `start_ts`, `end_ts`, `confirmed_at`.
- **Price/Bar:** `open`, `high`, `low`, `close`, `volume`, `atr`.
- **Footprint:** `direction`, `sequence_high`, `sequence_low`, `displacement_ratio`, `sweep_side`.
- **Lifecycle:** `state` (New/Unmitigated/Touched/Filled), `state_updated_at`.
- **IRR Inputs:** `ob_price`, `next_swing_price`, `liquidity_pool_id`, `mss_tf`.
- **HUD Format:** pre-rendered string for the glanceable HUD row.
- **Schema:** see `schemas/footprint.schema.json` for the canonical record contract.

Example (single emitted footprint record shape):

```
{
  "ticker": "AAPL",
  "underlier": "AAPL",
  "strike": 195,
  "expiry": "2025-01-17",
  "option_type": "call",
  "tf": "15m",
  "start_ts": "2024-11-21T14:30:00Z",
  "end_ts": "2024-11-21T14:45:00Z",
  "confirmed_at": "2024-11-21T14:45:00Z",
  "open": 2.35,
  "high": 2.68,
  "low": 2.28,
  "close": 2.61,
  "volume": 18420,
  "atr": 0.22,
  "direction": "bullish",
  "sequence_high": 2.68,
  "sequence_low": 2.21,
  "displacement_ratio": 0.74,
  "sweep_side": "high",
  "state": "New",
  "state_updated_at": "2024-11-21T14:45:00Z",
  "ob_price": 2.48,
  "next_swing_price": 2.94,
  "liquidity_pool_id": "aapl-2024-11-21-1420-high",
  "mss_tf": "1h",
  "hud_row": "AAPL | 195C | 2025-01-17 | 15m | Bull | New | 0.18"
}
```

## Confirmation Rules (Non-Negotiable)
- **Bar Finality:** a bar is final only at its closing timestamp; partials are never emitted.
- **Three-Candle Gate:** the footprint emits only after the third candle in the 3-bar sequence closes.
- **ATR Attachment:** ATR is computed and attached on the same timeframe as the emitted bar.
- **Deterministic Rounding:** prices and ratios are rounded consistently per instrument tick size to avoid replay divergence.

## Groundwork Artifacts
- `docs/groundwork.md` defines baseline deliverables, lifecycle expectations, and HUD contract.
- `schemas/footprint.schema.json` provides the JSON Schema for emitted footprint records.

## State Machine (Deterministic)
- **New → Unmitigated:** footprint recorded and awaiting first touch.
- **Unmitigated → Touched:** any overlap by current price action.
- **Touched → Filled:** full overlap of the gap; removed from active radar immediately.

## Latency and Delivery Expectations
- **Target Latency:** sub-1ms end-to-end from tick receipt to footprint emission (provider + tickerplant + validation).
- **Delivery Semantics:** at-most-once emission per footprint; no repainting updates.
- **Ordering:** events must be monotonic by `confirmed_at` within each instrument/timeframe.

---

## Implementation Note
The sources describe the architectural logic for low-latency ingestion and "A+ Setup" filtering but do not include the **sub-1ms technical implementation details** (e.g., C++ hardware-level optimizations or FPGA configurations). Verify hardware requirements independently when targeting sub-1ms execution on NASDAQ exchange infrastructure.

## Analogy
Developing this primitive is like installing a **high-precision sonar** on a high-speed submarine. The **NASDAQ exchange** is a vast, dark ocean. Without the sonar (the ingestion layer), you are navigating blindly, relying on storytelling or delayed information. This primitive sends out a constant ping (Data Ground Truth) that ignores schools of small fish (market noise) and only reports back when it detects a massive gold vein (an A+ Setup). It ensures the rest of the crew (the scoring engine and trader) only acts when the target is physically real and perfectly aligned.
