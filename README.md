# rpi-workflows

Cross-cutting [Swamp](https://github.com/swamp-club/swamp) workflows that
join data across the `@aaronge/rpi-*` Raspberry Pi extension family. This
repo publishes no extension of its own — it exists purely to host
workflows that pull already-published extensions and assert across them,
since a single workflow can only reference model instances registered in
its own repo.

## Pulled extensions and model instances

| Instance          | Type                          |
| -------------------- | -------------------------------- |
| `health-local`      | `@aaronge/rpi-health`            |
| `host-local`        | `@aaronge/host-health`           |
| `bootconfig-local`  | `@aaronge/rpi-boot-config`       |
| `cooling-local`     | `@aaronge/rpi-cooling`           |
| `pcie-local`        | `@aaronge/rpi-pcie`              |
| `apt-local`         | `@aaronge/apt-inventory`         |

Set up via:

```sh
swamp extension pull @aaronge/rpi-health
swamp extension pull @aaronge/host-health
swamp extension pull @aaronge/rpi-boot-config
swamp extension pull @aaronge/rpi-cooling
swamp extension pull @aaronge/rpi-pcie
swamp extension pull @aaronge/apt-inventory

swamp model create @aaronge/rpi-health health-local
swamp model create @aaronge/host-health host-local
swamp model create @aaronge/rpi-boot-config bootconfig-local
swamp model create @aaronge/rpi-cooling cooling-local
swamp model create @aaronge/rpi-pcie pcie-local
swamp model create @aaronge/apt-inventory apt-local
```

`apt-local` only ever runs `query` here (read-only, no privileges) — this
repo has its own, independent `apt-local` data separate from any
`apt-inventory` instance you might already be running elsewhere (e.g. as a
scheduled `swamp serve` service in the `apt-inventory` repo itself); the
two don't interact.

## Workflows

Every workflow's join is just multiple `data.latest("<instance>", "<name>")`
calls combined with CEL `&&`/`||` in one assert expression — no special
join primitive needed (confirmed against the Swamp docs and this family's
own experience building the individual extensions).

### `thermal-integrity`

If `health-local`'s `coreTempCelsius` exceeds `tempThresholdCelsius`
(default 60), `cooling-local`'s fan `rpm` must be greater than 0. Catches a
dead fan before it becomes a thermal event. Scheduled every 5 minutes.

### `power-integrity`

Two independent witnesses to the same underlying power problem:
`health-local`'s sticky `throttle.sinceBoot.underVoltage` must be `false`,
and its `pmic.supplyVolts` must exceed `minSupplyVolts` (default 4.8V). A
third witness — `smartctl`'s `unsafe_shutdowns` counter on the NVMe SSD,
the one most directly relevant given a ~4.6W drive on a ~5W-budget PCIe
slot — is **not yet included**: `smartctl` isn't installed and there's no
SSD on this machine as of writing. Add a `readSmart`-equivalent step and a
third assert clause once the NVMe migration happens. Scheduled every 5
minutes.

### `link-integrity`

Asserts `pcie-local`'s `linkDegraded == false`. Silent PCIe performance
loss made loud. This duplicates the assertion `@aaronge/rpi-pcie` already
ships in its own workflow — kept here too so it's visible alongside this
family's other cross-cutting checks in one place. Scheduled every 5
minutes.

### `pre-migration-snapshot`

**Not scheduled — run by hand.** One versioned record of a known-good
system: `bootconfig-local`'s `eeprom`/`configTxt`/`cmdline`, `host-local`'s
`filesystems`, `health-local`'s `firmware`, and `apt-local`'s `installed`
packages. Run once right before the NVMe SSD migration, then run again
after and diff the two versions — Swamp versions every write, so the diff
is free as long as the raw text fields are stored verbatim (they are).
Compare with `data.version("<instance>", "<name>", <old-version>)` against
whatever version number each `swamp data list <instance>` showed before
the migration.

```sh
swamp workflow run pre-migration-snapshot
```

## Scheduling

`thermal-integrity`, `power-integrity`, and `link-integrity` are inert on
their own — a workflow's `trigger.schedule` is just a declaration; it only
does something once *some* process actually reads and acts on it.

**This repo does not run a persistent `swamp serve` daemon.** An earlier
version of this setup did (one `swamp serve` process per repo across the
whole family, each holding its port open 24/7) — measured at ~400MB
resident per idle daemon, ~3GB combined across 8 repos on this 8GB Pi,
with swap already 65% utilized. Reverted the same day in favor of
`systemd.timer` + oneshot `swamp workflow run`, which costs ~0MB between
runs since each invocation is a plain process that starts, runs the
workflow (a couple of seconds), and exits:

```
# /etc/systemd/system/swamp-workflow-rpi-workflows-thermal-integrity.service
[Unit]
Description=swamp workflow run thermal-integrity (rpi-workflows)

[Service]
Type=oneshot
User=aaronge
Group=aaronge
Environment=HOME=/home/aaronge
WorkingDirectory=/home/aaronge/Repositories/rpi-workflows
ExecStart=/usr/local/bin/swamp workflow run thermal-integrity --repo-dir /home/aaronge/Repositories/rpi-workflows
```

```
# /etc/systemd/system/swamp-workflow-rpi-workflows-thermal-integrity.timer
[Unit]
Description=Timer for swamp workflow run thermal-integrity (rpi-workflows)

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
```

Same pattern for `power-integrity` (`OnCalendar=*:1/5`) and
`link-integrity` (`OnCalendar=*:2/5`) — **deliberately staggered a minute
apart**, not all at `*:0/5`, since all three read `health-local` and
firing simultaneously risks two separate workflow runs contending on its
model lock. `pre-migration-snapshot` has no timer at all — manual-only,
by design (`swamp workflow run pre-migration-snapshot`).

`systemctl enable --now swamp-workflow-rpi-workflows-{thermal,power,link}-integrity.timer`.
Verify with `systemctl list-timers 'swamp-workflow-rpi-workflows-*'`.

**`rpi-cooling` and `rpi-pcie` are deliberately not scheduled at all** —
their own checks (`stalled`, `linkDegraded`) are fully subsumed by
`thermal-integrity`/`link-integrity` here, which read the same underlying
data. Giving them their own timers too would just be redundant polling.
The rest of the family (`rpi-boot-config`, `rpi-connect`,
`rpi-gpio-inventory`, `rpi-camera`) each get their own
`swamp-workflow-<repo>.timer`/`.service` pair the same way, since none of
their scheduled content overlaps with what this repo covers.

The one exception to all of this on the machine as a whole is
`apt-inventory`, which keeps a persistent `swamp serve` daemon (port
9090) — its `apt-status`/`apt-monitor` toolkit depends on the live
WebSocket API. Nothing in this repo needs that, so it doesn't have one.

## License

MIT — see LICENSE for details.
