# SEL-551 Dummy Circuit Breaker Simulator

This repository contains the configuration files and documentation to repurpose an **SEL-551 overcurrent protection relay** as a simulated (dummy) circuit breaker. This simulator is designed to represent realistic circuit breaker behavior (opening and closing delays, dead-time block, and spring recharge block) during protection scheme testing.

## Folder Structure

```
DUMMY_BREAKER/
├── README.md             # This file (project overview and user guide)
├── GEMINI.md             # AI-Assisted design process and version log
├── settings/             # Active SEL-551 settings text files (V0.6)
│   ├── SET_1.TXT         # Global/Element settings (protection disabled, timers)
│   ├── SET_L.TXT         # SELOGIC control equations (state latch, logic)
│   ├── SET_T.TXT         # Display point texts and local control buttons
│   ├── SET_R.TXT         # Sequential Events Recorder (SER) selection
│   └── SET_P1.TXT        # Port 1 serial parameters
└── docs/                 # Reference guides and manuals
    ├── 551_IM_20250131.pdf               # SEL-551 Instruction Manual
    ├── DUMMY_BREAKER_PLAN.md             # Breaker logic design specifications
    └── DUMMY_BREAKER_TECHNICIAN_GUIDE.md # Technician connection and operation guide
```

---

## Features & Version History

- **v0.0:** Disables all overcurrent protection. Sets CTR=1. Establishes independent 100 ms trip (`SV5`) and close (`SV6`) timers to delay `TRIP` latch state changes. Maps `OUT1`/`OUT2` to breaker state (52A/52B).
- **v0.2:** Updates hardware part number and display labels.
- **v0.3:** Adds anti-pumping protection (`SV3` logic) and a 1 s reclose memory window (`SV7`).
- **v0.4:** Updates front-panel display points to dynamically reference settings timers (`SV5PU` and `SV6PU`) rather than hardcoding "100ms" text.
- **v0.5:** Consolidates trip and close delays into a single shared timer (`SV5`), avoiding split timer races. Gated by `IN1` for tripping (`TR = SV5T * IN1`) and close (`ULTR = SV5T * !IN1`).
- **v0.6 (Current):** 
  - Adds front-panel local controls **MAN TRIP (LB1)** and **MAN CLOSE (LB2)** soft switches.
  - Implements **0.3 s dead-time block** (`SV6`) to prevent closing within 300 ms of an open command.
  - Implements **15 s spring-charge recharge block** (`SV8`) following a Close-Open (CO) operation (tripping within 1 s of a close). Blocks closing and energizes `OUT3` (Amber Lamp) during recharge.

---

## Technical Specifications

### Input/Output Configuration

| Terminal | Function | Description |
|:---|:---|:---|
| **IN1** | External Trip Command | From Relay-Under-Test (RUT) trip contact or test set |
| **IN2** | External Close Command | From RUT close contact, test set, or external button |
| **OUT1** | 52A Contact (N/O) | Closed when breaker is CLOSED; Open when breaker is OPEN (Green Lamp) |
| **OUT2** | 52B Contact (N/C) | Closed when breaker is OPEN; Open when breaker is CLOSED (Red Lamp) |
| **OUT3** | Spring Charging | Energized for 15 s after a Close-Open (CO) operation (Amber Lamp) |
| **OUT4** | Operating | Energized during the 100 ms opening or closing delays (Amber Lamp) |
| **LB1** | Local Button 1 | Front-panel soft switch: `MAN TRIP` (Momentary) |
| **LB2** | Local Button 2 | Front-panel soft switch: `MAN CLOSE` (Momentary) |

### Control Panel Display

- **Top Line:** Always shows the breaker status: `CB OPEN` or `CB CLOSED` (controlled by `DP1`).
- **Bottom Line:** Shows `ANTI PUMP` if trip and close commands overlap while open, or `OP/CL DLY SV5PU` normally.

---

## Wiring Diagram

Below is the connection scheme to integrate the dummy breaker with a Relay Under Test (RUT):

```
                   SEL-551 DUMMY BREAKER
  ┌─────────────────────────────────────────────┐
  │                                             │
  │  IN1 (+) ◄─── Trip output from RUT         │
  │  IN1 (–) ◄─── DC common                    │
  │                                             │
  │  IN2 (+) ◄─── Close cmd (RUT / test set /  │
  │  IN2 (–) ◄─── push button) / DC common     │
  │                                             │
  │  OUT1 (+) ──► 52A to RUT / Green lamp       │
  │  OUT1 (–) ──► 52A common                   │
  │                                             │
  │  OUT2 (+) ──► 52B to RUT / Red lamp         │
  │  OUT2 (–) ──► 52B common                   │
  │                                             │
  │  OUT3 (+) ──► Amber lamp — SPRING CHG       │
  │  OUT4 (+) ──► Amber lamp — OPERATING        │
  └─────────────────────────────────────────────┘
```

---

## Adjusting Timing Parameters

Timers are configured in cycles (at 50 Hz, 1 cycle = 20 ms).

1. **Trip/Close Operating Delay:** Change `SV5PU` in `SET_1.TXT` (Default: `5.000` cycles = 100 ms).
2. **Reclose Dead Time:** Change `SV6PU` in `SET_1.TXT` (Default: `15.000` cycles = 300 ms).
3. **Spring Recharge Duration:** Change `SV8DO` in `SET_1.TXT` (Default: `750.000` cycles = 15 s).

For detailed operation and troubleshooting, refer to [DUMMY_BREAKER_TECHNICIAN_GUIDE.md](file:///M:/DUMMY_BREAKER/repo/DUMMY_BREAKER/docs/DUMMY_BREAKER_TECHNICIAN_GUIDE.md).
