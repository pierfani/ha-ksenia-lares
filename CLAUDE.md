# CLAUDE.md

Upstream's agent guidance applies here too — read and follow it:

@AGENTS.md

This file adds only what is specific to this **fork checkout** (fork workflow,
in-flight branches) plus architecture notes that upstream does not have yet.

## Fork & contribution workflow

This checkout is a **fork**. The remotes are:

- `origin` → `pierfani/ha-ksenia-lares` (personal fork; `main` tracks `origin/main`).
- `upstream` → `johnnybegood/ha-ksenia-lares` (the original repo, where releases live).

Contributions go upstream via **fork pull requests**:

1. Branch off **`upstream/main`**, not the fork's `main`: `main` carries this fork-only
   `CLAUDE.md`, which must not leak into upstream PRs (it happened once in #49).
2. Commit, then `git push -u origin <branch>` to the fork.
3. Open the PR against `upstream/main`. **`gh` CLI is not installed here** — open PRs via
   the GitHub "compare across forks" URL:
   `https://github.com/johnnybegood/ha-ksenia-lares/compare/main...pierfani:ha-ksenia-lares:<branch>?expand=1`
4. Link related issues in the PR body (`Fixes #NN` / `Closes #NN`).
5. To update an open PR with newer upstream changes, `git merge upstream/main` into the
   branch (history rewrites/force-pushes are blocked by the local permission settings).

Consequences to keep in mind:

- A fix lives only on its feature branch / PR until merged upstream — local `main` (and
  `origin/main`) will still show the **pre-fix** code. That's expected, not a lost change.
- After a PR is merged upstream, resync: `git fetch upstream && git merge upstream/main`
  (or `git pull upstream main`), then `git push origin main` to realign the fork.
- Upstream keeps its own `CLAUDE.md` as a short pointer to `AGENTS.md` (proposed in #49).
  When that lands, resolve the `CLAUDE.md` conflict on the fork's `main` by keeping this file.

## Tests

Upstream has a pytest suite (`tests/`, see `tests/README.md`). Local environments, both
gitignored: `.venv-current` (Python 3.14, current HA) and `.venv-minimum` (Python 3.12,
HA 2024.12). Run e.g. `.venv-current/bin/python -m pytest`. CI also runs hassfest and the
HACS action on every PR; neither runs locally.

## In-flight branches (pending upstream merge)

These live on the fork (`origin`); they are **not** on `main`/`upstream/main` yet, so `main`
still shows the pre-change code (expected). When a PR is merged upstream, resync `main` and
delete its row here.

- **`feature/output-control`** (PR #48, `Closes #44`) — panel **outputs (automations)** as
  `switch` entities: `output_descriptions()` + `outputs()` + `set_output()` in `base.py`,
  outputs fetched in `coordinator.py` with their own timeout so a failure leaves zones/partitions
  updating, `LaresOutputSwitch` in `switch.py` (raises on rejected command / missing PIN), reuses
  the options `CONF_PIN`, adds `translations/it.json`. **Gotcha:** `setOutput` uses a
  **0-indexed** `outputId` (`outputId = idx`), unlike `setByPassZone` which is 1-indexed —
  verified against a 128IP panel.
- **`feature/configurable-scan-interval`** (PR #49) — polling interval configurable via the
  options flow (`scan_interval`, 5–300 s, default 10 s); keeps the saved PIN when the field is
  left empty; moves the architecture notes into `AGENTS.md` and makes upstream's `CLAUDE.md` a
  pointer. Conflicts with #48 (`coordinator.py`, `strings.json`, `translations/*`,
  `tests/README.md`): whichever merges second needs `upstream/main` merged and resolved.
- **`feature/https-host-url`** (PR #59) — the host field accepts a full URL
  (`https://…`, optional port), e.g. for an HTTPS reverse proxy. Touches `base.py` and strings.
- **`integration/test`** — local only, never pushed: all of the above merged together for
  testing on a real HA instance.

## Architecture (until #49 adds it to AGENTS.md)

The integration polls the panel's **HTTP web interface**: XML documents under
`http://<host>:<port>/xml/...` with HTTP Basic Auth. `local_polling`, no push. The panel's
web interface must be enabled by the installer (often disabled by default).

1. **[base.py](custom_components/ksenia_lares/base.py) — `LaresBase`**: the only thing that
   touches the panel. Every read is `get(path)` → parsed `lxml` tree, or `None` on any error
   (errors are swallowed and logged at debug). Writes go through `send_command()`
   (`cmd/cmdOk.xml?cmd=...&pin=<code>&...`); never log that URL.
2. **[coordinator.py](custom_components/ksenia_lares/coordinator.py)** — a `DataUpdateCoordinator`
   polling `zones()` + `partitions()` every 10 s. Its `.data` dict is the single source of truth.
3. **Platform entities** — all `CoordinatorEntity` subclasses from the one config entry: a
   motion `binary_sensor` per zone (on when `ALARM`), an ENUM `sensor` per partition, one
   `alarm_control_panel`, a bypass `switch` per zone.
4. **[config_flow.py](custom_components/ksenia_lares/config_flow.py)** — setup (host / port
   `4202` / username / password) + options flow (state→scenario/partition mapping, bypass PIN).

### HA states ↔ Ksenia scenarios + partitions

- **Arming uses a *scenario*** (`CONF_SCENARIO_*`): the configured name is resolved to an index
  in `scenario_descriptions()` and passed to `activate_scenario(idx, code)`. A state is only
  advertised when its scenario is configured.
- **Reading state uses *partitions*** (`CONF_PARTITION_*`): a state applies when *all* its
  mapped partitions are armed; otherwise `ARMED_CUSTOM_BYPASS`.

Arm/disarm use the code typed in the HA UI per action; zone bypass uses the PIN stored in
options (`CONF_PIN`) and silently no-ops (error log) without it.

### Conventions & gotchas

- `get_model()` → `"16IP"`/`"48IP"`/`"128IP"`, interpolated into zone/partition XML filenames
  (e.g. `zonesStatus48IP.xml`); scenario files are not model-suffixed.
- Entities are keyed by **positional index** (`lares_zones_{idx}`, `lares_partitions_{idx}`,
  `lares_bypass_{idx}`); reordering on the panel shifts entities. `bypass_zone()` sends
  `zoneId = idx + 1` (panel zones are 1-indexed).
- `NOT_USED` zones/partitions are created but disabled+hidden by default.
- Description lists are cached on `LaresBase`; status lists are re-fetched every poll.
- Device unique ID = MAC (via `getmac`), falling back to `host:port`.
- `async_migrate_entry` (v1→v2) backfills `port=4202`; bump `ConfigFlow.VERSION` and extend it
  for any future schema change.
