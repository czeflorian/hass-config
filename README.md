# Alpstuga Dashboard

## Hardware

- **HA Connect ZBT-2** — Thread border router (Matter-over-Thread devices:
  GRILLPLATS plug, BILRESA dual-button remote)
- **Sonoff Zigbee dongle** — Zigbee coordinator (lights, shades, sensors)
- **No DIRIGERA hub** — IKEA Matter devices pair directly via the ZBT-2

If the ZBT-2 dies and is replaced, Matter devices need re-pairing (Thread
credentials live on the radio). If the Sonoff dongle dies, the Zigbee
coordinator backup (Z2M: `coordinator_backup.json`, ZHA: `zigbee.db`) lets
a same-model replacement take over without re-pairing.

## Architecture

```
config/
├── configuration.yaml              # references packages dir
├── automations.yaml                # GUI-editable, single file
├── scripts.yaml                    # GUI-editable, single file
├── packages/
│   ├── status_sensors.yaml         # all the *_status template sensors
│   └── shade_automation.yaml       # shade helpers + sensors + status
├── themes/
│   └── misty_forest.yaml           # comprehensive theme
└── dashboards/home/
    ├── home.yaml                   # main view, includes sections
    ├── decluttering_templates.yaml # reusable card definitions
    └── sections/
        ├── living_room.yaml
        ├── kitchen_bedroom.yaml
        └── energy_appliances.yaml
```

## Design principle: logic in the backend, display in the dashboard

Every threshold, state mapping, or color decision lives in a template
sensor in `packages/`. Each emits a normalized status string (`good`,
`moderate`, `poor`, `low`, `hazardous`, `active`, `idle`, `unavailable`)
that the dashboard maps to a color via the standard `status_tile` family
of decluttering templates.

To add a new state-aware tile:
1. Define a `*_status` template sensor in the relevant package file.
2. Use one of the `status_tile` / `status_tile_attr` / `status_tile_text`
   templates in the dashboard.

The dashboard YAML stays declarative — no inline Jinja for color choices.

## Why automations and scripts aren't split into directories

Home Assistant's GUI editor only edits the *default* automation/script files
(`automations.yaml`, `scripts.yaml` at the config root). The moment you split
them via `!include_dir_merge_list automations/` or similar, those entries
become read-only in the GUI — you can see and trigger them, but the pencil
icon disappears.

The GUI editor is good (visual trigger picker, condition builder, action
autocomplete) and the cost of giving it up is higher than the cost of having
one large automations file. So: **single-file, GUI-editable** for automations
and scripts.

Dashboards, template sensors, and decluttering templates are **not**
GUI-editable in YAML mode regardless of where they live, so we split those
freely for maintainability.

## Required configuration.yaml entries

```yaml
# Template sensors via packages
homeassistant:
  packages: !include_dir_named packages

# Default automation and script files (these are normally already present)
automation: !include automations.yaml
script: !include scripts.yaml

# Lovelace YAML mode (only required for the structured dashboard)
lovelace:
  mode: yaml
  dashboards:
    lovelace-home:
      mode: yaml
      filename: dashboards/home/home.yaml
      title: Home
      icon: mdi:home-heart
      show_in_sidebar: true
```

## Design principles

**Logic lives in the backend, not the dashboard.** Threshold decisions
(what counts as "good" PM2.5, "low" battery) are template sensors. The
dashboard reads the resulting `_status` sensor and maps it to a color via a
single decluttering template. To change a threshold, edit one place in
`packages/air_quality.yaml` — never the dashboard.

**Visual repetition is solved by `decluttering-card`.** Every recurring
card pattern (status tile, plain tile, light tile, cover tile, section
title, air quality block) is a template in `decluttering_templates.yaml`.
A new room's air quality section is *one card invocation*, not 80 lines of
copy-pasted Jinja.

**Colors live in the theme, nowhere else.** The dashboard references
`var(--color-good)`, `var(--color-warn)`, `var(--color-bad)`. No hex
values in dashboard YAML. Switch theme = whole dashboard restyles.

**Unavailable states are explicit.** Every template sensor and card
checks `unavailable`/`unknown`/`none` before doing arithmetic. Sensors that
are dead show "Offline" with a disabled-color icon, not "0%" in green.

## Adding a new room

1. Add three template sensors to `packages/air_quality.yaml` (AQI, CO₂,
   humidity status) — copy a block, change the entity prefix.
2. In a section file, add:

   ```yaml
   - type: custom:decluttering-card
     template: air_quality_block
     variables:
       - prefix: alpstuga_office              # entity prefix
       - prefix_clean: office                  # status sensor prefix
   ```

That's it.

## Adding a new threshold-driven tile

1. Add a `*_status` template sensor that emits `good`/`moderate`/`poor`.
2. Use the `status_tile` decluttering template:

   ```yaml
   - type: custom:decluttering-card
     template: status_tile
     variables:
       - entity: sensor.something
       - status_entity: sensor.something_status
       - label: Something
       - icon: mdi:something
       - unit: " units"
   ```

## Known issues / TODOs

- **Energy entity ID** (`sensor.wnsm_at...`): verify against your wnsm
  integration's actual exposed entity in Developer Tools → States.
- **`waschemasche` typo** is in the entity IDs themselves (set by the
  Miele integration during pairing). Rename via Settings → Devices →
  entity → ⚙ → entity_id, then update sensor references in this dashboard
  with find-and-replace.
- **Mobile layout**: this is desktop-first. On narrow viewports the
  horizontal-stacks of three light cards become cramped. A mobile view
  would use `vertical-stack` for those rows, or a separate dashboard.
