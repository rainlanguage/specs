# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## What this repo is

This is a **specification repository** for the Rain protocol ecosystem. It
contains primarily markdown documents — no application source code and no tests.
A `flake.nix` provides a Nix dev shell via rainix. Each `.md` file is a
standalone spec defining a protocol, format, or subsystem.

## Spec documents

- **rainlang.md** — Language spec for Rainlang, an onchain interpreted
  expression language
- **dotrain.md** — `.rain` file format: a composable container for Rainlang
  fragments, 1:1 with cbor-seq metadata
- **metadata-v1.md** — Binary metadata format (magic number prefix, cbor-seq
  encoding, onchain events)
- **raindex-yaml.md** — YAML configuration format for Raindex strategies
  (tokens, orders, deployments, scenarios, charts, etc.)
- **gui.md** — GUI configuration spec: how `raindex-yaml` deployments are
  presented as interactive UI elements (bindings, deposits, presets)
- **cas.md** — Content Addressable Storage for `.rain` imports (hash-based,
  location-independent)
- **gas.md** — Analysis of gas costs for an onchain interpreted language vs
  native Solidity
- **router.md** — Route-finding library spec for onchain liquidity (arb-bot
  component)
- **raindex-ide-errors.md** — Error pipeline design for the Raindex "Add Order"
  UI
- **i9r/sub-parser-registry.md** — Sub-parser registry to replace opaque
  addresses with human-readable names
- **meta/web-data-v1.md** — Minimal format for specifying offchain data URLs
  (8-byte API magic + UTF-8 URL)
- **meta/st0x-spec.md** — st0x server API: signed price quotes for Raindex order
  solvers

## Conventions

- Specs use RFC 2119 language (MUST, SHOULD, MAY) where precision matters.
- YAML examples in specs (especially `raindex-yaml.md` and `gui.md`) are
  normative — they define the schema.
- The `raindex-yaml.md` spec follows explicit design principles: minimal, avoid
  repetition, prefer named k/v sets over lists, flat hierarchies, strict YAML
  parsing, mergeable structures.
