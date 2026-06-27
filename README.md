# midea-cli

An interactive terminal CLI for **Midea (NetHome Plus) WiFi air conditioners**.
Control is **local** over your LAN — the cloud is only touched once, during
setup, to fetch the unit's local key.

It gives you:

- 🎛️ **Direct control** — power, target temp, mode, fan speed, live status
- 🌡️ **A software thermostat** (`smart`) — cycles the unit on/off to hold a room
  temperature, with a deadband + compressor short-cycle protection
- ⏲️ **Auto-off timers** — `timer 30m`, `timer 22:30`, or set-mode-and-arm in one go
- 📈 **In-terminal charts** — a background poller logs outdoor/home/target temps
  and `chart` plots them right in your shell
- ⌨️ **A proper REPL** — arrow-key history and line editing, persisted across runs

```text
$ midea
Connected.
midea> smart 25
Smart mode ON — holding home ≤ 25°C (on at ≥25.5, off at ≤25).
  14:02:13 OFF  home 24.6°C · target 25°C · out —
midea> chart 6
```

## Install

Requires Python 3.10+.

### With pipx (recommended)

```bash
pipx install git+https://github.com/your-username/midea-cli
midea
```

### From source

```bash
git clone https://github.com/your-username/midea-cli
cd midea-cli
python3 -m venv .venv
.venv/bin/pip install -e .
.venv/bin/midea
```

Or run the single file directly without installing:

```bash
.venv/bin/pip install -r requirements.txt
.venv/bin/python midea.py
```

## First run

On first launch it scans the LAN for your AC and saves its
`ip`/`id`/`token`/`key` to `config.json` (next to the module). If your region
needs your own login, re-run and enter your NetHome Plus email/password when
prompted, or set `NETHOME_ACCOUNT` / `NETHOME_PASSWORD` / `AC_REGION` env vars.

`config.example.json` shows the shape of that file — you normally don't write it
by hand; the first run generates it for you.

## Commands

| Command         | Action                                            |
|-----------------|---------------------------------------------------|
| `on` / `off`    | power the unit on/off                              |
| `temp 23`       | set target temperature                            |
| `mode cool`     | mode: `cool` `heat` `auto` `dry` `fan`            |
| `fan auto`      | fan: `auto` `low` `medium` `high` `max` `silent`  |
| `status`        | show current state                                |
| `chart [hours]` | plot outdoor / home / target temp (default 6h)    |
| `timer 30m`     | turn off after a duration (`30m`/`1h30m`) or `HH:MM` |
| `timer cool 23 30m` | set mode+temp and power on now, then off after that time |
| `timer cancel`  | cancel a pending timer                            |
| `smart 25`      | software thermostat: on above 25.5°C, off at 25°C  |
| `smart 25 0.3`  | same with a custom 0.3°C deadband                 |
| `smart off`     | disable smart mode                                |
| `poll 30`       | change background sampling interval (seconds)      |
| `quit`          | exit                                              |

## The chart

A background poller samples the unit every `poll_interval` seconds (default 60)
and appends outdoor/home/target temps to `history.csv`. `chart` plots that
history in the terminal. The longer the app runs, the richer the chart.

> Outdoor temperature is read from the AC's outdoor unit sensor. Most Midea
> units report it (sometimes coarse / a degree or two off), and typically only
> while the unit is powered on; a few don't report it at all, in which case that
> line will be absent.

## The timer

`timer` schedules an **auto-off**. The timer lives in the running app, so keep
it open until it fires (it's cancelled if you `quit`). On firing it powers the
unit off and logs a sample.

- `timer 30m` — off in 30 minutes
- `timer 1h30m` — off in 90 minutes
- `timer 22:30` — off at 22:30 today (or tomorrow if already past)
- `timer cool 23 45m` — switch to cool @ 23°C and power on now, off in 45 min
- `timer cancel` — clear it

## Smart mode (software thermostat)

`smart <temp>` turns the app into an on/off thermostat: every 30s it reads the
home temperature and

- powers **ON** (cool @ target) when home ≥ `temp + deadband` (deadband 0.5°C),
- powers **OFF** when home ≤ `temp`.

While it's running it echoes the readings live on the line above the prompt, so
you don't have to keep typing `status`. The deadband stops it chattering around
the setpoint, and a **180s minimum** between switches protects the inverter
compressor from short-cycling. Smart mode owns the power state while active (a
manual `on`/`off` may be reverted on the next check) — `smart off` or `quit` to
release it.

> Note: this is bang-bang on/off control. For an *inverter* unit, simply setting
> a target and letting it modulate (`mode cool` + `temp 25`) is usually gentler
> and more efficient — smart mode is for when you specifically want it to fully
> stop once the room is cool. Because the unit's temperature reading is smoothed
> (see below), expect a little overshoot past the setpoint.

## Notes

- **AUTO mode** auto-selects heating vs cooling to hold the target; it does
  *not* power the unit off at the setpoint. `Power ON` means energized, not
  necessarily actively cooling.
- Some models report no power/energy data, so there's no "compressor running"
  readout — only temperatures and the power flag.
- Reported temps (especially outdoor) are smoothed/updated slowly by the unit's
  firmware, so they can trail reality by several minutes. The chart's *time*
  axis is accurate; the *values* lag at the source.
- If the AC's IP changes, delete `config.json` and re-run to rediscover.
- Give the unit a reserved/static IP in your router to keep it stable.
- `config.json` holds your device key — it's gitignored; don't commit it.

## Contributing

Issues and PRs welcome. It's a single module (`midea.py`) plus
[`msmart-ng`](https://github.com/mill1000/midea-msmart),
[`rich`](https://github.com/Textualize/rich), and
[`plotext`](https://github.com/piccolomo/plotext), so it's easy to hack on.

## Disclaimer

This is an **unofficial** tool, not affiliated with or endorsed by Midea. It
relies on the reverse-engineered local protocol via `msmart-ng`; behavior varies
by model and firmware, and a vendor update could break it. Use at your own risk.

## License

[MIT](LICENSE) © 2026 Adi Fatol
