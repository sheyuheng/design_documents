# 2026_4_27 Janus4K AS Review Disposition

Sources reviewed:

- `Janus4k_AS_CN.md`
- `Janus4k_AS_EN.md`

Formal files updated:

- `../Janus4K/Janus4k_AS.md`
- `../Janus4K/Janus4k_AS_EN.md`

## Accepted Into Formal AS

| Area | Accepted Material |
|---|---|
| Structure | Added the outerCube-style table-of-contents shape and kept the implementation-facing section order. |
| Output Buffer | Kept entry fields, lock/writeback protocol, forwarding visibility, reduction residency, and writeback arbitration rules. |
| TileReg Arbitration | Kept RR read arbitration, 2 KB grant unit, Global Src Buffer, and write-port arbitration guarantees. |
| Interfaces | Kept valid/ready rule, BCC/PE, read request, Output Buffer, TMA, uOp queue, and TileReg writeback tables. |
| Scheduling | Kept two-level TileOp/uOp scheduling, wakeup types, dispatch state machine, and resolve/retire protocol. |
| Backpressure | Kept the backpressure table and the no-drop rule for TileOp/uOp/data/tag. |
| Verification | Kept BCC, Vector, TMA/Cube, pipeline/backpressure tests, and verification principles. |

## Not Absorbed

| Material | Decision | Reason |
|---|---|---|
| Physical floorplan recommendations | Discarded as formal constraints. | Not backed by floorplan, macro, clocking, routing, or PPA data. |
| Expected bottleneck ranking | Kept out of the formal spec. | Requires workload and implementation data. |
| Potential port-waste claim | Kept out as a conclusion. | Utilization depends on workload and scheduler policy. |
| Version comparison appendix | Kept out. | Historical note, not architecture contract. |
| Python parameter appendix | Kept out. | Parameter table is authoritative; implementation files can be generated later. |

## Remaining Open Items

- Exact BCC instruction encoding and error reporting.
- Exact TileReg address-to-bank mapping.
- Output Buffer physical capacity, CAM/tag organization, lock granularity, and release rule.
- uOp queue depths and selection rules.
- Cube array/L0/accumulator/output-layout details.
- dtype/packing/mask Tile encoding.
