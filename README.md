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

`thermal-integrity`, `power-integrity`, and `link-integrity` are inert
without a `swamp serve` process running against this repo — a workflow's
`trigger.schedule` is only registered at server startup, not evaluated
standalone. Running as a systemd service, `127.0.0.1:9093` only (no
network exposure needed for a local schedule):

```
# /etc/systemd/system/swamp-serve-rpi-workflows.service
[Unit]
Description=swamp serve (rpi-workflows)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=aaronge
Group=aaronge
Environment=HOME=/home/aaronge
WorkingDirectory=/home/aaronge/Repositories/rpi-workflows
ExecStart=/usr/local/bin/swamp serve --repo-dir /home/aaronge/Repositories/rpi-workflows --port 9093
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`systemctl enable --now swamp-serve-rpi-workflows.service`. Verified via
`journalctl -u swamp-serve-rpi-workflows.service`: all 3 scheduled
workflows registered (`Scheduled execution service started with 3
schedules`) and `pre-migration-snapshot` correctly absent from that count
(it has no `trigger.schedule` — manual-only, as intended).

**`rpi-cooling` and `rpi-pcie` are deliberately not served on their own**
— their own scheduled checks (`stalled`, `linkDegraded`) are fully
subsumed by `thermal-integrity`/`link-integrity` here, which read the same
underlying data. Running both would just be two processes polling
identical `vcgencmd`/sysfs calls for equivalent checks. The rest of the
family (`rpi-boot-config`, `rpi-connect`, `rpi-gpio-inventory`,
`rpi-camera`) each run their own `swamp-serve-<repo>.service` on ports
9094-9097 respectively, since none of their scheduled content overlaps
with what this repo covers.

Port map on this machine: `apt-inventory` 9090 · `host-health` 9091 ·
`rpi-health` 9092 · `rpi-workflows` 9093 · `rpi-boot-config` 9094 ·
`rpi-connect` 9095 · `rpi-gpio-inventory` 9096 · `rpi-camera` 9097.

## License

MIT — see LICENSE for details.
