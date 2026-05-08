# outerCube AS Style Guide

## Source Corpus

Use the immutable GitHub commit corpus as the style source:

- Directory: `https://github.com/hengliao1972/pyCircuit/tree/cbfc7c5341f104a7c453d6dd6be744da7f4f1881/designs/outerCube`
- Raw base: `https://raw.githubusercontent.com/hengliao1972/pyCircuit/cbfc7c5341f104a7c453d6dd6be744da7f4f1881/designs/outerCube`
- Main files: `outerCube.md`, `vector4k.md`, `tregfile.md`, `tregfile4k.md`, `Davinci_supersclar.md`

## Style Fingerprint

- English, implementation-facing AS style with literal block-name titles.
- Heavy use of numbered sections and subsections.
- Dense Markdown tables for parameters, data formats, physical organization, ports, opcodes, latencies, calendars, and mode comparisons.
- Fenced code blocks are common for ASCII architecture diagrams, cycle traces, bitfield sketches, command examples, and pseudocode.
- Mermaid can be used, but ASCII diagrams are the default for hardware datapaths and timing.
- The text is specific about units, geometry, protocol behavior, and current constraints.
- Worked examples are part of the style, especially for epoch-aligned schedules, striping, cross-lane movement, reductions, legal format/shape combinations, and bypass rules.

## Corpus Patterns

`outerCube.md` emphasizes:

- `Overview & Key Features`
- `Key Parameters`
- comparison tables
- top-level block diagrams
- data formats and MAC scaling
- architecture and dual-mode operation
- interfaces
- epoch-aligned and execution pipelines
- instruction set

`vector4k.md` emphasizes:

- purpose and scope
- tile and format model
- TRegFile interface and striping
- vector datapath overview
- epoch-aligned fiber calendars
- instruction categories and cycle sketches
- cross-lane / cross-strip summaries
- implementation notes
- legal `(format, R x C)` enumeration and axis-reduce complexity

`tregfile4k.md` emphasizes:

- core parameters
- tile layout and physical organization
- port interface
- synchronized calendar
- throughput
- write-to-read bypass and scheduling constraints

`Davinci_supersclar.md` emphasizes:

- overview and design philosophy
- ISA summary
- top-level block diagram
- pipeline overview
- front-end, decode, rename, dispatch, issue, execute, retire details
- many tables tying resources, widths, structures, and timing together

## Canonical AS Skeleton

````markdown
# <Block Name> Design

## 1. Overview & Key Features

### 1.1 Key Parameters

| Parameter | Value | Notes |
|---|---:|---|

### 1.2 Top-Level Block Diagram

```text
<ASCII block diagram>
```

## 2. Data Formats / Tile Model

| Format | Element Bits | Elements / Tile | Notes |
|---|---:|---:|---|

## 3. Architecture

## 4. Operation Modes

| Mode | Use Case | Datapath | Constraint |
|---|---|---|---|

## 5. Interfaces

| Signal | Direction | Width | Rate | Description |
|---|---|---:|---|---|

## 6. Pipeline / Epoch Calendar

```text
cycle:  0 1 2 3 4 5 6 7
stage:  ...
```

## 7. Instruction Set / Command Format

| Opcode | Fields | Latency | Description |
|---|---|---:|---|

## 8. Throughput and Latency

## 9. Implementation Notes and Verification
````

Adjust section names for the target block, but preserve the contract-first ordering.

## Table Patterns

Use these tables whenever the corresponding facts exist:

- `Parameter | Value | Notes`
- `Port | Direction | Width | Count | Aggregate Bandwidth | Notes`
- `Signal | Direction | Width | Valid/Ready | Description`
- `Field | Bits | Type | Description`
- `Opcode | Operands | Fiber/Axis | Latency | Notes`
- `Format | Element Bits | R x C | Elements/Tile | Legal | Notes`
- `Stage | Cycle | Input | Operation | Output | Stall Condition`
- `Mode | Enabled Datapath | Shared Resource | Constraint`
- `Hazard | Trigger | Required Stall/Bypass | Verification Case`

Prefer one table with exact units over multiple prose paragraphs.

## Diagram and Timing Patterns

Use fenced `text` blocks for stable monospace layout:

```text
      cmd_valid/cmd_ready
              |
              v
        +-----------+
        | Scheduler |
        +-----+-----+
              |
   +----------+----------+
   |                     |
+--v---+             +---v--+
| Lane | ...         | Lane |
+------+             +------+
```

For cycle timing, keep the calendar compact and include the unit of work:

```text
epoch e, fiber f

cycle:        0       1       2       3       4       5       6       7
TReg read:    f0      f1      f2      f3      f4      f5      f6      f7
execute:      -       f0      f1      f2      f3      f4      f5      f6
writeback:    -       -       f0      f1      f2      f3      f4      f5
```

Always state what happens on stall, FIFO full, invalid data, bypass hit, or epoch boundary.

## Terminology to Preserve

Use project terms consistently:

- `TRegFile`, `TRegFile-4K`
- `4 KB tile`
- `lane`, `bank`, `strip`, `fiber`, `fiber_id`
- `epoch`, `epoch-aligned`, `calendar`
- `valid/ready`, `FIFO`, `backpressure`
- `ping-pong`, `bypass`, `hazard`
- `Mode A`, `Mode B` when describing dual-mode behavior
- `opcode`, `operand`, `command`, `latency`, `throughput`
- `MAC`, `accumulator`, `reduction`, `cross-lane`, `cross-strip`

## Review Checklist

- Does the document start with a clear block role and key parameters?
- Are all dimensions and bandwidths stated with units?
- Are tile, format, lane, bank, and strip mappings unambiguous?
- Are read/write ports and aggregate bandwidth explicit?
- Are valid/ready and backpressure semantics specified?
- Are pipeline stages tied to cycles or epochs?
- Are hazards, bypass, and scheduling constraints explicit?
- Are command/opcode fields and legal combinations tabulated?
- Are throughput and latency derived from stated parameters?
- Are worked examples included for non-obvious schedules or reductions?
- Are current limitations and verification cases listed?
