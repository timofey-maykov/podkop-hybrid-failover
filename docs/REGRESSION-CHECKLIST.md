# Hybrid Failover Core: regression checklist

Run in QEMU lab or on router after installing `hybrid-failover-core`.

## Automated (host: no router)

```sh
chmod +x ./scripts/regression-smoke.sh
./scripts/regression-smoke.sh
# or on router: HF_BIN=/usr/sbin/hybrid-failover ./scripts/regression-smoke.sh
```

Covers: binary build, `help`, **all** `examples/testdata/*.conf` validate (dry-run), `go test ./...`, `go test ./internal/migrate/...`, optional `sing-box check`.

CI: [`.github/workflows/go-ci.yml`](../.github/workflows/go-ci.yml): sing-box **1.12.4+**, smoke on push/PR; optional QEMU on `workflow_dispatch`.

## QEMU lab (semi-automated)

```sh
chmod +x ./scripts/regression-qemu.sh ./scripts/openwrt-qemu-lab.sh
./scripts/regression-qemu.sh                    # build + lab + remote + luci + smoke
./scripts/regression-qemu.sh build smoke      # host-only
./scripts/regression-qemu.sh remote luci      # remote checks + ubus smoke
./scripts/regression-qemu.sh remote failover  # remote checks + failover poll
```

QEMU lab installs **`hybrid-failover-core`** + **`luci-app-hybrid-failover`** (not legacy patch). Remote step: migrate, start, check-nft, global-check, pending validate, optional check-fakeip, `ubus call hybrid-failover status` + `export_history`. **`luci`** step runs `scripts/luci-ubus-smoke.sh` on the guest.

**Manual gate (not in CI):** AWG kernel module, real proxy URIs, LAN client tproxy under load, VPN interface down failover.

---

## Install

- [x] `hybrid-failover-core` package has no runtime dep on `jq`, `python3-light` (see `packages/hybrid-failover-core`)
- [ ] `apk add` on OpenWrt 25+ router/QEMU: run `./scripts/regression-qemu.sh lab remote`
- [ ] `/etc/init.d/hybrid-failover enable && start`: verified in `regression-qemu.sh remote`

## Config / migration

- [x] `hybrid-failover migrate` sets `config_schema_version=1` and `cache_path` (`internal/migrate/migrate_test.go`)
- [x] postinst runs migrate ([`packages/hybrid-failover-core/CONTROL/postinst`](../packages/hybrid-failover-core/CONTROL/postinst))
- [x] Legacy failover hooks trigger warning in `hybrid-failover migrate` (`warnLegacyScripts`)
- [ ] UCI path `/etc/config/hybrid-failover` on deployed router

## sing-box

- [x] `hybrid-failover validate` on fixtures (regression-smoke)
- [x] `apply --dry-run` via validate on fixtures (regression-smoke)
- [ ] `apply && start` generates `/etc/sing-box/config.json` on router: `regression-qemu.sh remote`
- [x] `sing-box check` on generated configs (CI, sing-box ≥ 1.12.4)
- [x] Minimum sing-box version enforced in `lifecycle.Apply` (`internal/singbox/version.go`)
- [x] VPN + failover outbounds: golden + structural tests
- [x] urltest duration validation tests

## Network

- [ ] nft table `inet hybrid_failover` after start: `regression-qemu.sh remote` (`check-nft`)
- [ ] LAN client tproxy (port 1602): **manual**
- [ ] dnsmasq restored after stop: **manual** (`hybrid-failover stop`)
- [x] per-client rules idempotent refresh on `reload` (`internal/perclient`, `lifecycle.RefreshPerClient`)

## DNS / FakeIP

- [ ] dnsmasq → `127.0.0.42` after start: `regression-qemu.sh remote` (when outbound UCI configured)
- [ ] `check-fakeip` on router with sing-box up: `regression-qemu.sh remote`

## Amnezia

- [x] `vpn://` decodes without python3 (`internal/amnezia` tests)
- [ ] `awg2://` with kernel module: **manual**

## Lists / subscription

- [x] `list-update` triggers apply + reload when config changes (`runListUpdate` + `ApplyAndReloadIfChanged`)
- [ ] `list-update` fetches lists on router: **manual** (needs network + sing-box up)
- [ ] `subscription-refresh` merges URIs: **manual**

## Diagnostics / bot

- [x] `global-check` skips fakeip when sing-box down; checks when running (`internal/diag`)
- [ ] Clash API reachable on router: `regression-qemu.sh remote`
- [ ] Bot `/health` via core RPC: **manual**
- [x] pending validate / apply / rollback (CLI + bot + LuCI rpcd)

## LuCI

- [ ] Services pages on router: **manual** (QEMU LuCI after lab install)
- [x] Dashboard polls `ubus hybrid-failover status` + `history` (`dashboard.js`)
- [x] Routing: save → `pending_capture`, Apply → `pending_apply`, Rollback (`routing.js` + rpcd)
- [x] Clients: save → `reload` + per-client nft refresh (`clients.js` + `RefreshPerClient`)
- [x] ubus smoke: `scripts/luci-ubus-smoke.sh` (status, health, history, export_history): `regression-qemu.sh luci`
- [x] rpcd ACL: switch_proxy, decode_uri, backup_uci, restore_uci, duplicate_section

## Failover

- [ ] VPN down → urltest backup: **manual**
- [ ] Event in `history.jsonl`: `regression-qemu.sh failover` (best-effort Clash poll)
