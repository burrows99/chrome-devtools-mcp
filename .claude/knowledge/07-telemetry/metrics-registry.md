# Metrics Registry, Transformation & Error Codes

> **Layer:** Telemetry & logging · **Sources:** `src/telemetry/metricsRegistry.ts`, `src/telemetry/transformation.ts`, `src/telemetry/errors.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Two related concerns: (1) **`transformation.ts`** sanitizes tool params at runtime so no raw values leave the machine; (2) **`metricsRegistry.ts`** is a **build-time codegen** helper that produces a stable schema (`tool_call_metrics.json`) describing which metrics each tool *could* emit. `errors.ts` holds the `ErrorCode` enum.

## Key files
- `src/telemetry/transformation.ts` — runtime param sanitization + bucketization + name/type transforms
- `src/telemetry/metricsRegistry.ts` — build-time tool-metric schema generation (run by `scripts/update_metrics.ts`)
- `src/telemetry/errors.ts` — the `ErrorCode` enum (intentionally minimal)

## Key types & functions
**transformation.ts (runtime):**
- `sanitizeParams(params, schema)` — the privacy gate; converts a params object into safe metric values (used by `ClearcutLogger.logToolInvocation`).
- `bucketizeLatency(ms)` — snaps latency to `[50,100,250,500,1000,2500,5000,10000]` (called in `ToolHandler`).
- `stripUnderscoreBeforeNumber(name)` — collapses `_<digit>` → `<digit>` (e.g. `tool_2` → `tool2`), keeping names protobuf/metric-name friendly. Used widely.
- `getZodType`, `transformArgName`, `transformArgType`, `PARAM_BLOCKLIST` — shared by both files.

**metricsRegistry.ts (build-time):**
- `generateToolMetrics(tools)` — turns `ToolDefinition[]` into `ToolMetric[]` (name + arg names/types).
- `applyToExisting` / `applyToExistingMetrics` — merge new metrics into existing JSON, preserving order and **marking removed entries `isDeprecated: true`** rather than deleting them.
- `validateEnumHomogeneity(values)` — enums must be all-one-primitive-type.

## How it works

### Param sanitization (the privacy mechanism)
`sanitizeParams` iterates params and, per `PARAM_BLOCKLIST = {uid, reqid, msgid}`, skips those entirely. For the rest it resolves the zod type (unwrapping `ZodOptional`/`ZodDefault`/`ZodNullable`/`ZodEffects`) and transforms:
- **Strings:** name → `<snake>_length`, value → `bucketize(value.length)` over `[0,1,2,5,10,20,50,100,200,500,1000,2000,5000,10000]`. The raw string is never sent.
- **Arrays:** name → `<snake>_count`, value → `.length` (raw count, not bucketized).
- **Number / boolean / enum:** name + value passed through; enums kept as-is.
`hasEquivalentType` throws if the runtime value's type doesn't match the declared zod type. Only `ZodString/Number/Boolean/Array/Enum` are supported — anything else throws (`Unsupported zod type`).

### Build-time schema generation
`generateToolMetrics` mirrors the same name/type transforms (via `transformArgName`/`transformArgType`) but produces *descriptors* (`{name, argType}`), skipping blocklisted params and resolving enums to their homogeneous primitive type. `scripts/update_metrics.ts` (run via `npm run update-metrics`, part of `npm run gen`) builds full + slim tool sets, merges with the existing `src/telemetry/tool_call_metrics.json`, and writes it back. It also validates `ErrorCode` is sequentially numbered from 0 with no `_<digit>` patterns.

### Error codes
```ts
export enum ErrorCode {
  ERROR_CODE_UNSPECIFIED = 0,
  ERROR_CODE_PERSISTENCE_FILE_READ_FAILED = 1,
  ERROR_CODE_PERSISTENCE_FILE_SAVE_FAILED = 2,
}
```
The file header mandates: it must contain *only* `ErrorCode`, no refactor elsewhere, and new values must be prefixed `ERROR_CODE_` (so the file can be parsed programmatically). Currently only persistence emits these.

## Relationships
- **Depends on:** `src/third_party` (zod types), `ToolDefinition` ([tools](../02-tools/README.md)) for the build-time path.
- **Used by:** [ClearcutLogger](clearcut-logger.md) (`sanitizeParams`, `stripUnderscoreBeforeNumber`), [ToolHandler](../02-tools/tool-handler.md) (`bucketizeLatency`), `scripts/update_metrics.ts` (codegen), [persistence](flags-and-persistence.md) (`ErrorCode`).

## Gotchas & non-obvious details
- **Two different bucket arrays:** latency buckets vs. the param-length `BUCKETS` array — don't conflate them.
- Array params log a **raw count**, not a bucketized one (unlike string length).
- Metrics JSON is **append-only by convention**: removed tools/args become `isDeprecated`, never dropped, to keep historical metric IDs stable server-side.
- `transformArgType` maps both `ZodString` and `ZodArray` to `'number'` (their logged metric is a length/count number).
- `sanitizeParams` *throws* on type mismatch or unsupported zod type; the call in `ClearcutLogger` is `void`-ed and any throw is swallowed by the caller's fire-and-forget pattern.

## Update triggers
- `PARAM_BLOCKLIST`, the bucket arrays, or supported zod types change.
- A new `ErrorCode` is added (must stay sequential).
- The codegen contract (`generateToolMetrics` / `tool_call_metrics.json` shape) changes.
