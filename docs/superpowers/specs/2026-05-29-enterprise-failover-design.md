# Enterprise failover (2026-05-29)

## Topology (outage-only / prefer-primary)

- `{section}-awg-out`: primary VPN (selector **default**)
- `{section}-urltest-out`: urltest on **backup proxies only**
- `{section}-out`: selector: primary | urltest | individual backups

## Policy controller

Background loop (`internal/failover/controller.go`, 30s):

1. Live probe primary via Clash `/delay`
2. On primary: 2 failed probes → switch selector to `urltest-out`
3. On backup: when primary OK (2× outage-only, 1× prefer-primary) → switch back to `awg-out`
4. History + webhook on each switch

## UCI `failover_policy`

| Value | Behavior |
|-------|----------|
| `outage-only` | Default; no switch while VPN probe OK |
| `prefer-primary` | Faster failback to VPN |
| `fastest` | Legacy: all channels in one urltest |

State file: `/var/run/hybrid-failover/policy-state.json`
