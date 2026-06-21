# Architecture Decision Records

Short, append-only records of the significant choices behind this project
("Pocket Dev" — ephemeral LazyVim + Copilot CLI on a Raspberry Pi Zero 2 W).

| ADR | Title | Status |
|-----|-------|--------|
| [0001](0001-access-model-usb-gadget.md) | Access model: USB gadget (CDC-ECM) + SSH | Accepted |
| [0002](0002-host-choice.md) | Host hardware choice (Zero 2 W primary) | Proposed (deferred to Phase 1 bench) |
| [0003](0003-local-console-first.md) | Local console with no desktop environment, validated first | Accepted |

Format: one file per decision, named `NNNN-kebab-title.md`. Never rewrite an
accepted ADR's decision; supersede it with a new one instead.
