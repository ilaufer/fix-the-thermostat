# Fix the Thermostat: Development Plan

## The problem

The house has a **Honeywell VisionPRO 8000 WiFi** thermostat (likely model TH8320WF),
controlled through the **Total Connect Comfort (TCC)** app.

Colorado weather swings a lot in a single day: below 60°F at night, above 70°F in the
afternoon. Right now someone has to switch the system between **Heat** and **Cool** by
hand as the day goes on.

**Goal:** switch between heating and cooling automatically, based on the **indoor**
temperature and the **outdoor** temperature (now and forecast).

---

## Phase 0: Try the built-in fix first (no code)

The VisionPRO 8000 has an **Auto Changeover** feature built in, but installers often
leave it turned off. When it's on, the thermostat holds both a heat setpoint and a
cool setpoint and picks whichever one it needs.

- [x] On the thermostat: **Menu → Installer Options → Installer Setup**
- [x] Find **ISU 300 (System Changeover)** and set it to **Automatic**
- [ ] Set the **Auto Changeover Deadband**: the minimum gap between the heat and cool
      setpoints, 2–9°F, default 3°F. Example: heat 68°F, cool 74°F.
- [ ] Set the system mode to **Auto** (on the thermostat or in the TCC app)
- [ ] Run it for about a week and keep notes *(started 2026-10-04)*

**Why do this first?** It might solve most of the problem for free, and it runs on the
thermostat itself, so it keeps working when the internet is down. What it can't do is
look at the weather. It only reacts to the indoor temperature, so it can't tell that
"it's 55°F outside now, but it'll hit 80°F by noon, so don't bother heating."
Phases 1–5 add that kind of smarter logic on top.

> ⚠️ Before turning on Auto, check that your HVAC system really has both heating and
> cooling wired to this thermostat (it should, since you switch between them today).

---

## The software idea

A small program that runs on a schedule (say, every 10 minutes) and does this:

```
 ┌──────────────┐     ┌──────────────┐
 │ Weather API  │     │  TCC cloud   │
 │ (outdoor now │     │ (indoor temp,│
 │  + forecast) │     │ mode, setpts)│
 └──────┬───────┘     └──────┬───────┘
        │   read             │ read / write
        ▼                    ▼
     ┌───────────────────────────┐
     │      Decision engine      │  ← your rules + comfort settings
     │  "should we heat, cool,   │
     │   or do nothing?"         │
     └─────────────┬─────────────┘
                   ▼
        set mode / setpoints on thermostat
        + log what happened and why
```

### Main building blocks

| Piece | What it does | Likely tool |
|---|---|---|
| **Thermostat client** | Log in to TCC, read indoor temp, mode and setpoints; change mode and setpoints | [`aiosomecomfort`](https://github.com/mkmer/aiosomecomfort) (Python, unofficial; the same library Home Assistant uses) |
| **Weather client** | Current outdoor temp + hourly forecast | [Open-Meteo](https://open-meteo.com/) (free, no API key) or the US National Weather Service API |
| **Decision engine** | Pure logic: inputs → "heat / cool / off / leave alone" | Our own Python code (easy to unit-test) |
| **Scheduler / runner** | Runs the loop every N minutes | Raspberry Pi or an always-on computer at home, or a small cloud VM |
| **Config** | Comfort temps, thresholds, location, credentials | A `config.yaml` + a `.env` file for secrets (never committed) |
| **Logging / history** | Record every reading and decision | CSV or SQLite to start |

**Language:** Python. It's beginner-friendly, and the TCC library is already in Python.

---

## Decision logic: first draft

Start simple. "Hysteresis" just means a buffer zone, so the system doesn't flip back and
forth every few minutes.

```
comfort_low  = 68°F    # below this indoors → we want heat
comfort_high = 74°F    # above this indoors → we want cool

if indoor < comfort_low:
    want HEAT
    BUT if the forecast says it'll be > 75°F outside within ~2 hours
        and indoor is only slightly low → skip it (the sun will do the work)
elif indoor > comfort_high:
    want COOL
    BUT if it's cool outside right now (< 60°F) and we're only slightly warm → skip it
       (later: suggest opening windows)
else:
    leave it alone

Safety rules:
  - Don't switch modes more than once every 30–60 minutes (protects the HVAC equipment)
  - Never set temps outside a hard safe range (e.g. 55–85°F)
  - If anything fails (no internet, TCC login fails) → do nothing; the thermostat keeps
    its last setting
  - Respect manual overrides: if a person changed the thermostat recently, back off for
    a few hours
```

Later ideas: different comfort ranges for day/night/away, using sun and solar heating,
pre-cooling before a hot afternoon, humidity.

---

## Phases and milestones

### Phase 0: Built-in Auto Changeover *(above)*
**Done when:** you've tried it for about a week and know what it doesn't handle well.

### Phase 1: Project setup + "can we talk to the thermostat?"
- [ ] Python project skeleton (`pyproject.toml`, `src/`, `tests/`)
- [ ] `.env` for TCC username/password, plus `.gitignore` so secrets are never committed
- [ ] A script that logs in and **prints** the indoor temp, mode and setpoints (read-only)
- [ ] A script that **changes** one setting (e.g. the heat setpoint by 1°F) and changes it back

**Done when:** we can read and control the thermostat from code.
*Biggest risk:* TCC is an unofficial, undocumented API. Honeywell rate-limits it and
sometimes changes it. Test it early so there are no surprises later.

### Phase 2: Weather data
- [ ] Get current outdoor temp + 12-hour hourly forecast for your location from Open-Meteo
- [ ] Print it alongside the indoor data

**Done when:** one command shows indoor + outdoor + forecast together.

### Phase 3: Data logging ("observe before you automate")
- [ ] Run a read-only loop every 10 min that logs indoor temp, outdoor temp, mode,
      setpoints and whether HVAC is running
- [ ] Let it collect a few days of data (ideally including a big-swing day)
- [ ] Make a simple chart: indoor vs. outdoor over a day

**Done when:** we have real data from your house to tune the rules against.
*Why:* it shows how fast the house heats up or cools down, so we pick thresholds from
real numbers instead of guesses.

### Phase 4: Decision engine (dry-run mode)
- [ ] Write the rules as a pure function: `decide(indoor, outdoor, forecast, state, config) → action`
- [ ] Unit tests for the important cases (cold morning, hot afternoon, mild day, API failure…)
- [ ] Run the loop in **dry-run**: log what it *would* do, but don't touch the thermostat
- [ ] Compare its decisions with what you would have done

**Done when:** you agree with the dry-run decisions for a few days.

### Phase 5: Go live
- [ ] Turn on real control (a config flag: `dry_run: false`)
- [ ] Add the safety rules: minimum time between switches, safe temp limits, backing off
      after manual changes
- [ ] Deploy to an always-on machine and start it automatically on boot
- [ ] Alerts: a notification if the program crashes or can't reach TCC for a while

**Done when:** a full big-swing day goes by and nobody touches the thermostat.

### Phase 6: Nice-to-haves (pick later)
- Simple web dashboard or phone notifications ("switched to cooling at 11:40, it's 72°F inside")
- Schedules (night / day / away comfort ranges)
- "Open your windows" suggestions when it's nice outside
- Smarter forecasting (learning how fast your house warms and cools)
- Home Assistant integration

---

## Alternative path: Home Assistant

[Home Assistant](https://www.home-assistant.io/) is free home-automation software that
already has a **Honeywell TCC integration** plus weather integrations. You could build
the same "if outdoor/indoor X then switch mode" rules with its automation editor and
write little or no code.

| | Custom Python app (this plan) | Home Assistant |
|---|---|---|
| Learning to code | ✅ Great learning project | Less code, more configuring |
| Setup effort | Medium | Medium (needs a Pi or always-on box) |
| Flexibility | Total | High, but complex logic gets awkward |
| Extras | Build what you want | Dashboards, phone app, 1000s of integrations |

Our recommendation: do **Phase 0** now, then decide. If you want a coding project,
follow this plan. If you just want it fixed, Home Assistant may get there faster.

---

## Open questions for you

1. **Hardware:** do you have an always-on computer at home, or a Raspberry Pi? Would you
   buy one (about $50–80)? Or would you rather run it in the cloud?
2. **Comfort range:** what temps do you like? (e.g. "never below 67°F, never above 75°F")
3. **Schedule:** different temps at night or when nobody's home?
4. **Equipment:** furnace + AC? Heat pump? (This changes the safety rules a little.)
5. **Is Auto Changeover already on?** Check Phase 0 and let us know what you find.

---

## Risks and notes

- **Unofficial API:** TCC has no public API. `aiosomecomfort` reverse-engineers the
  website. It works today, but Honeywell could break it. Poll gently (every 5–10 min
  at most) to avoid rate limits.
- **Credentials:** your TCC password goes in a local `.env` file and never in git.
- **Equipment safety:** switching between heat and cool too often is hard on the
  equipment. The minimum time between switches matters.
- **Fail safe:** if our program dies, the thermostat simply keeps its last setting.
  Nothing dangerous happens.

## References

- VisionPRO 8000 Auto Changeover (ISU 300) and deadband: [Resideo product data](https://customer.resideo.com/resources/techlit/TechLitDocuments/33-00000s/33-00096.pdf), [TH8320WF installation manual](https://www.manualslib.com/manual/1177247/Honeywell-Th8320wf.html?page=5)
- TCC Python library: [mkmer/aiosomecomfort](https://github.com/mkmer/aiosomecomfort)
- Weather: [Open-Meteo](https://open-meteo.com/), [NWS API](https://www.weather.gov/documentation/services-web-api)
