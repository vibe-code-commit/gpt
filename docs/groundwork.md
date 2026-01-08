# System Groundwork

This groundwork document defines the baseline deliverables for the **Data Oracle and Ingestion Layer** so implementation can proceed without ambiguity.

## Deliverables

1. **Ingestion Interface Contract**
   - Emit a single, validated footprint record per detected sequence.
   - Records must satisfy the confirmation rules and locked definitions in `readme.md`.
   - Payload structure is defined by the JSON Schema in `schemas/footprint.schema.json`.

2. **State Machine Baseline**
   - Lifecycle states: `New → Unmitigated → Touched → Filled`.
   - Transition logic is deterministic and should be evaluated on each confirmed bar close.

3. **Latency Targets & Ordering**
   - Sub-1ms end-to-end target from tick receipt to footprint emission.
   - Event ordering must be monotonic by `confirmed_at` per instrument/timeframe.

4. **HUD Output Contract**
   - Each record emits a pre-rendered HUD row with aligned columns:
     `Ticker | Strike | Expiry | TF | Zone | Status | Distance`.

## Implementation Notes

- This groundwork is intentionally provider-agnostic: the feed may be OPRA directly or a low-latency vendor.
- Hardware-specific optimizations are outside the scope of this document and must be validated separately.
