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

## License

MIT — see LICENSE for details.
