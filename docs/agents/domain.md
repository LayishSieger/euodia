# Domain docs

How skills should consume this repo's domain documentation.

## Before exploring, read these

- **`CONTEXT.md`** at the repo root, or
- **`CONTEXT-MAP.md`** at the root if it exists (points at per-context `CONTEXT.md` files)
- **`docs/adr/`** — ADRs that touch the area you are changing

If these files do not exist, proceed without blocking. Domain docs can be added later.

## Layout

**Single-context** (default):

```text
/
├── CONTEXT.md
└── docs/adr/
```

**Multi-context**:

```text
/
├── CONTEXT-MAP.md
├── docs/adr/
└── src/<context>/CONTEXT.md
```

## This repo

- Layout: **single-context** _(or multi-context — edit after init)_
