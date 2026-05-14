# BCC Diagram Sources

> DOT is used for architecture / structure / flow diagrams.  
> WaveDrom JSON is used for timing diagrams.

## DOT Diagrams

| File | Diagram |
|------|---------|
| [bcc_top.dot](bcc_top.dot) | JCore BCC top architecture |
| [block_header_dispatch.dot](block_header_dispatch.dot) | Block header dispatch flow |
| [tilerename_bisq.dot](tilerename_bisq.dot) | TileRename and BISQ dependency resolution |
| [brob_resolve_commit.dot](brob_resolve_commit.dot) | BROB resolve and commit flow |

Render examples:

```powershell
dot -Tsvg E:\Workarea\design_documents\BCC\diagrams\bcc_top.dot -o E:\Workarea\design_documents\BCC\diagrams\bcc_top.svg
dot -Tpng E:\Workarea\design_documents\BCC\diagrams\block_header_dispatch.dot -o E:\Workarea\design_documents\BCC\diagrams\block_header_dispatch.png
dot -Tsvg E:\Workarea\design_documents\BCC\diagrams\tilerename_bisq.dot -o E:\Workarea\design_documents\BCC\diagrams\tilerename_bisq.svg
dot -Tsvg E:\Workarea\design_documents\BCC\diagrams\brob_resolve_commit.dot -o E:\Workarea\design_documents\BCC\diagrams\brob_resolve_commit.svg
```

## WaveDrom Timing Diagrams

| File | Diagram |
|------|---------|
| [block_header_pipeline.wavedrom.json](block_header_pipeline.wavedrom.json) | Block header rename / dispatch timing |
| [resolve_commit_timing.wavedrom.json](resolve_commit_timing.wavedrom.json) | BROB resolve vs commit timing |
| [cmd_isq_tilerename_timing.wavedrom.json](cmd_isq_tilerename_timing.wavedrom.json) | CMD_ISQ / TileRename backpressure timing |

WaveDrom CLI examples vary by local installation. With `wavedrom-cli`, the command shape is typically:

```powershell
wavedrom-cli -i E:\Workarea\design_documents\BCC\diagrams\resolve_commit_timing.wavedrom.json -s E:\Workarea\design_documents\BCC\diagrams\resolve_commit_timing.svg
```
