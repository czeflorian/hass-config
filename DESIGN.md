# DESIGN

Architectural decisions and the reasoning behind them. Read this before
making structural changes.

## North star

**Logic in the backend, display in the dashboard.** Anywhere the dashboard
makes a decision based on values (thresholds, state mapping, color choice),
that decision belongs in a template sensor. The dashboard reads the result
and renders it. This keeps dashboard YAML declarative and lets us reuse
display patterns via `decluttering-card` templates.

The smell to watch for: multi-line Jinja inside `icon_color:` or `secondary:`
on a card. If you see that, the logic should probably move to a template
sensor.

## Status vocabulary

Every `*_status` template sensor emits one of these strings. The
`status_tile` family of decluttering templates maps them to colors:

| Status | Color (var) | Use for |
|--------|-------------|---------|
| `good` | `--success-color` | Healthy, in-range |
| `active` | `--success-color` | Running, on, doing its job |
| `moderate` | `--warning-color` | Edge of acceptable, needs attention |
| `poor` | `--error-color` | Out of range |
| `low` | `--error-color` | Resource depleted (battery, ink) |
| `hazardous` | `--error-color` | Dangerous (extreme PM2.5) |
| `idle` | `--secondary-text-color` | Off, standby, nothing happening |
| `unavailable` | `--disabled-text-color` | Sensor dead/missing |

When adding a new status sensor, normalize to this vocabulary. Don't invent
new states unless genuinely needed (and if you do, extend the templates).

## File organization

```
packages/                # template sensors, helpers, binary sensors
  status_sensors.yaml    # all the *_status sensors used by the dashboard
  shade_automation.yaml  # shade-specific helpers + sensors

automations.yaml         # all automations
scripts.yaml             # all scripts
configuration.yaml       # entry point, includes everything

dashboards/home/         # the home dashboard
  home.yaml              # view definition, includes sections
  decluttering_templates.yaml  # reusable card patterns
  sections/              # per-area dashboard files
```

## Decluttering templates

Three templates handle ~95% of state-aware tiles:

- **`status_tile`** — Use when the displayed value is the entity's raw state
  with optional unit suffix. Color comes from a separate `*_status` sensor.

- **`status_tile_attr`** — Use when one status sensor encodes everything:
  state value drives the color, `text` and `icon` attributes drive the
  display. Good for state-machine entities (washing machine, plug).

- **`status_tile_text`** — Use when secondary text is custom-formatted but
  the color still comes from a status sensor. Good for cards combining
  multiple sensors into one display string (e.g. "Black: 80% · Color: 75%").

There's also `plain_tile` for raw values without status logic, and domain-
specific composites: `light_tile`, `cover_tile`, `air_quality_block`,
`smart_plug_block`.

## Automation patterns

### Manual override detection

We DO NOT use `trigger.to_state.context.parent_id` to distinguish
automation-initiated changes from user-initiated ones. That pattern is
fragile and broke in HA 2025.7+.

Instead, an automation that commands a device:
1. Sets `input_boolean.<feature>_automation_moving = on`
2. Issues the command
3. Delays 30 seconds (covers settle, devices report final state)
4. Sets the moving flag back to off

The detection automation only fires when the moving flag is off. Bulletproof.

### Status sensor template

```yaml
- name: "Foo Status"
  unique_id: foo_status
  state: >
    {% set v = states('sensor.foo') %}
    {% if v in ['unknown', 'unavailable', 'none'] %} unavailable
    {% else %}
      {% set v = v | float %}
      {% if v < 30 %} good
      {% elif v < 60 %} moderate
      {% else %} poor
      {% endif %}
    {% endif %}
```

Always check `unavailable`/`unknown`/`none` BEFORE doing arithmetic. Never
use `int(0)` or `float(0)` as a fallback for missing data — that lies.

## Theme

Misty Forest defines a comprehensive set of HA core variables (~130 per
mode) with named YAML anchors at the top. Color the dashboard by referencing
standard HA semantic variables (`--success-color`, `--warning-color`,
`--error-color`, `--info-color`) rather than hardcoded hex.

The theme also remaps `mush-rgb-blue` to teal — when a Mushroom card is
configured with `icon_color: blue`, it picks up the warm teal instead of
HA's default blue. This avoids the cool-blue-vs-warm-palette clash.

## What's deliberately NOT done

- **Multi-file automation directories.** Keeps GUI editability. (Note: the
  user is moving to full YAML control as of v8 — see migration plan.)
- **Frontend module CSS injection.** Brittle, breaks on HA updates.
- **Cross-instance Thread credential restore.** Documented in README — it's
  a Matter limitation, not something we can fix here.

## Things on the radar

- BILRESA double-press / long-press bindings (currently single-press only)
- Mobile-optimized layout (currently desktop-first)
- Phantom-trigger filtering on cover override detection (only an issue if
  observed in practice — don't preemptively complicate)
