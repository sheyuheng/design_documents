---
name: janus4k-tyle
description: Write, restructure, or review Janus4K AI hardware Architecture Spec (AS) Markdown in the outerCube, VEC-4K, TRegFile-4K, and Davinci style from hengliao1972/pyCircuit designs/outerCube. Use for AS work under D:/Workarea/AS when the user asks for Janus4K-tyle, to match outerCube AS style, draft Cube/vector/tile-register/datapath/pipeline/ISA specs, or make an AS implementation-facing with dense parameter tables, interfaces, timing calendars, and worked examples.
---

# Janus4K-tyle

## Operating Mode

Use this skill when drafting, rewriting, or reviewing AS documents that should follow the outerCube corpus style. The target is an implementation-facing hardware spec: precise enough for RTL, pyCircuit, verification, and performance analysis.

Default to the user's requested language. If the user writes in Chinese, Chinese narrative is acceptable, but preserve English headings when they are part of the local AS convention and keep signal names, opcodes, formats, module names, and protocol terms in English.

## Workflow

1. Inspect the local target document or directory first. Prefer existing AS conventions in `D:/Workarea/AS` when they conflict with generic style advice.
2. Read `references/outercube-style-guide.md` before substantial drafting or review.
3. Identify the block type: Cube/MAC array, vector unit, tile register file, OoO core, command/control, memory/data movement, or mixed subsystem.
4. Build the spec around concrete contracts: parameters, formats, interfaces, cycle timing, valid/ready behavior, instruction or command fields, constraints, and verification points.
5. Use `TBD` only for facts that cannot be inferred safely; keep each `TBD` in the table or section where the implementer will resolve it.

## Required Shape

For a new or substantially rewritten AS, prefer this section order unless the local document already has a stronger structure:

1. `Overview & Key Features` or `Purpose and Scope`
2. `Key Parameters`
3. `Data Formats`, `Tile Model`, or `Register Model`
4. `Architecture` / `Datapath Overview`
5. `Interfaces`
6. `Operation Modes`
7. `Pipeline`, `Epoch Calendar`, or `Cycle Timing`
8. `Instruction Set`, `Command Format`, or `Programming Model`
9. `Throughput`, `Latency`, and `Performance Model`
10. `Implementation Notes`, `Scheduling Constraints`, `Debug`, and `Verification`

Every non-trivial block should have a top-level ASCII block diagram, at least one parameter table, interface tables, timing/cycle examples, and explicit current-version limitations.

## Writing Rules

- Prefer tables over prose for parameters, formats, legal shapes, port lists, latency, throughput, command fields, and mode differences.
- Use fenced code blocks for ASCII diagrams, cycle traces, calendars, bitfields, pseudocode, and instruction encodings.
- Be exact with units: bytes/cycle, bits, lanes, banks, strips, rows, columns, cycles, ops/cycle, and tile size.
- Include equations when behavior depends on geometry or bandwidth.
- Include worked examples when scheduling, epoch alignment, striping, reduction, bypass, or cross-lane behavior is not obvious.
- State valid/ready, FIFO, backpressure, bypass, hazard, and completion semantics explicitly.
- Keep implementation language concrete: "reads", "writes", "broadcasts", "accumulates", "stalls", "drains", "commits".
- Avoid marketing language, generic benefits, and high-level summaries that do not constrain implementation.

## Review Rules

When reviewing an AS for outerCube style, report issues in this priority order:

1. Missing or inconsistent interface, command, opcode, format, or bitfield contracts.
2. Ambiguous pipeline timing, epoch alignment, scheduling calendars, bypass behavior, hazards, or completion rules.
3. Tables, formulas, examples, and diagrams that contradict each other.
4. Missing legal shape enumeration, throughput/latency model, implementation notes, debug hooks, or verification cases.
5. Style drift: too much prose, too few tables, missing diagrams, missing cycle examples, or unclear units.

For each issue, give the affected section, why it blocks implementation or verification, and the concrete table/text/diagram to add.

## References

- `references/outercube-style-guide.md` - source corpus, style fingerprint, section skeletons, table patterns, timing patterns, terminology, and review checklist.
