# Hybrid Failover Core: design (2026-05-28)

## Goal

Standalone OpenWrt routing stack: Go binary `hybrid-failover`, **no dependency** on `jq` or `python3-light`.

## Components

| Piece | Role |
|-------|------|
| `hybrid-failover` | CLI + RPC: UCI → sing-box JSON, nft, dnsmasq, lists, failover controller |
| `hybrid-failover-bot` | Telegram control via RPC |
| `luci-app-hybrid-failover` | LuCI UI |

## Config

- UCI package: `hybrid-failover`
- File: `/etc/config/hybrid-failover`
- One-time migration: if only legacy UCI file exists on disk, `hybrid-failover migrate` copies it to `hybrid-failover`.

## Operations

| Command | Purpose |
|---------|---------|
| `migrate` | Import legacy UCI, schema v1 |
| `validate` / `apply` / `start` / `stop` | Lifecycle |
| `list-update` | Community rulesets |
| `subscription-refresh` | Pull proxy URIs from subscription URLs |
| `rpc *` | LuCI / bot |

## Packaging

Default release: `hybrid-failover-core`, `hybrid-failover-bot`, `luci-app-hybrid-failover`.

## First boot

`hybrid-failover migrate` imports legacy UCI if present, sets schema v1 (`cache_path`, failover defaults).

Warns if conflicting routing binaries or failover hooks are still installed.
