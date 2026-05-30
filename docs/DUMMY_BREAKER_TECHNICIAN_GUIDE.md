# SEL-551 Dummy Circuit Breaker — Technician Guide

**Relay:** SEL-551  
**Settings version:** DUMMY_CB_V0.6 (see `DUMMY_CB_V0.6/` folder)  
**System frequency:** 50 Hz

---

## 1. What is a Dummy Circuit Breaker?

When testing a protection relay with an Omicron test set (or similar), the test set injects fault current and times how long the relay takes to assert its trip output. The test set cannot instantly stop current as a real breaker would — a real circuit breaker takes 50–150 ms to open after receiving a trip command.

This SEL-551 is configured to simulate that breaker opening and closing delay. The relay-under-test and test set see it as a real circuit breaker, with correct timing behaviour on both trip and close.

**What the dummy breaker does:**
- Monitors **IN1** (or front-panel **MAN TRIP**) for a trip command
- After a **100 ms delay**, opens its 52A contact (OUT1) — simulating the breaker opening
- Monitors **IN2** (or front-panel **MAN CLOSE**) for a close command
- After a **100 ms delay**, closes its 52A contact (OUT1) — subject to 0.3 s dead time and spring charge
- Simulates the standard HV breaker operating sequence: **O — 0.3 s — CO — 15 s — CO**

The opening and closing delays use the same timer setting (`SV5PU`). Both are always equal.

---

## 2. Hardware Requirements

- SEL-551 relay (base model: 2 inputs IN1, IN2; 4 outputs OUT1–OUT4)
- DC control power supply (as per relay nameplate — typically 24 Vdc, 48 Vdc, 110 Vdc, or 125 Vdc)
- No current transformer (CT) or voltage transformer (VT) connections required
- Optional: external indicator lamps wired to OUT1–OUT4

---

## 3. Wiring

### 3.1 Control Power

Connect DC control power to the relay power supply terminals as per the relay nameplate. Observe correct polarity.

### 3.2 Input Connections

The SEL-551 optoisolated inputs IN1 and IN2 require control voltage to assert. The voltage must match the relay's input voltage rating (typically 24–250 Vdc).

**IN1 — Trip Command Input**

```
Relay-Under-Test TRIP output (normally open contact)
    ──┬── (+) DC supply ──── IN1 (+) terminal
      │                      IN1 (–) terminal ──── (–) DC supply / common
      └── Test set monitoring output (optional, in parallel)
```

When the relay-under-test closes its trip output contact, IN1 asserts and the dummy breaker starts its 100 ms opening timer.

**IN2 — Close Command Input**

Connect ONE or MORE of the following in parallel:

```
a) Relay-under-test CLOSE output (if autorecloser or close output exists)
b) Test set CLOSE output / timing channel output
c) External push button (normally open, momentary or maintained)

Any of the above: closed contact ──── (+) DC ──── IN2(+) ──── IN2(–) ──── (–) DC
```

When IN2 asserts, the dummy breaker starts its 100 ms closing timer — **provided the 0.3 s dead time has expired and the spring charge block is not active**.

### 3.3 Output Connections

**OUT1 — Breaker Status 52A (primary output) / Green lamp — CB CLOSED**

Normally open dry contact. **CLOSED** (makes) when the breaker is **CLOSED**; **OPEN** (breaks) when the breaker is **OPEN**.

```
Relay-under-test 52A input circuit:
    (+) Supervision voltage ──── OUT1 (+) ──── OUT1 (–) ──── 52A input (–)
```

Note: On the connectorised SEL-551, OUT1 is polarity-dependent. Terminal A02 must be (+) relative to A01.

**OUT2 — Breaker Status 52B / Red lamp — CB OPEN**

Complement of OUT1. **CLOSED** when breaker is **OPEN**; **OPEN** when breaker is **CLOSED**. Connect to the relay-under-test's 52B input if required.

**OUT3 — Spring Charge Indicator / Amber lamp — SPRING CHG**

Energised for 15 s after a Close–Open (CO) operation. The close path is blocked while OUT3 is active. OUT3 extinguishing indicates the spring is ready and close is permitted.

**OUT4 — Operating Indicator (optional) / Amber lamp — OPERATING**

Energised during the 100 ms opening or closing delay. Use to drive an amber lamp or capture a timing signal on the test set.

### 3.4 Wiring Diagram (Simplified)

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

RUT = Relay Under Test
```

---

## 4. Loading Settings

Settings files are in the `DUMMY_CB_V0.6/` folder:

```
DUMMY_CB_V0.6/
  SET_1.TXT   — Relay element settings (all protection OFF, timers set)
  SET_L.TXT   — SELOGIC control equations
  SET_T.TXT   — Display labels and local control button names
  SET_R.TXT   — Sequential Events Recorder
  SET_P1.TXT  — Serial port (19200 baud, 8N1)
```

### Loading via ACSELERATOR QuickSet

1. Open ACSELERATOR QuickSet.
2. Open the `SET_1.TXT` file (or import the folder as a settings group).
3. Connect to the relay via the serial port.
4. Send all settings groups (1, L, T, R, P1).
5. Confirm no errors are reported.

### Loading via Serial Terminal (HyperTerminal / PuTTY)

1. Connect at 19200 baud, 8N1.
2. Log in with access level 2 password.
3. Use `SET 1`, `SET L`, `SET T`, `SET R` commands to enter settings from each file.
4. Use `SHO 1`, `SHO L` to verify settings after entry.

---

## 5. Normal Operating Sequence

### Power-On

On power-up the TRIP latch initialises to 0. The dummy breaker starts in the **CLOSED** state:
- OUT1 energises (contact closes) → relay-under-test sees breaker CLOSED
- Front-panel TRIP LED is off
- Display shows:

```
┌────────────────┐
│ CB CLOSED      │  ← top row: breaker state
│ OP/CL DLY SV5PU│  ← bottom row: reminder of the delay setting name
└────────────────┘
```

### Performing a Trip Test

```
Test Step                    Dummy Breaker Response            Display
─────────────────────────    ─────────────────────────────     ──────────────────────
1. Test set injects fault    —                                 CB CLOSED / OP/CL DLY SV5PU
2. RUT trips (asserts OUT)   IN1 asserts                       CB CLOSED / OP/CL DLY SV5PU
                             Opening timer starts (100 ms)
3. After 100 ms:             OUT1 opens (breaker OPEN)         CB OPEN / OP/CL DLY SV5PU
                             TRIP LED illuminates
4. Test set detects OUT1     Test set stops injection
   opening — records time
```

The test set should monitor **OUT1** as the "breaker open" signal. Total tripping time from fault injection to breaker opening = relay-under-test protection time + 100 ms dummy breaker opening delay.

### Resetting After a Trip Test

1. If the relay-under-test has a **latching trip output**, reset it first so IN1 de-asserts.
2. Wait for the 0.3 s dead time to expire (SV6T=1). For simple tests, this is transparent.
3. Assert IN2 (press close button or have test set assert close command), or press **MAN CLOSE** on the front panel.
4. After 100 ms the breaker closes.

### Front-Panel Manual Reset

If IN2 is unavailable, press the **TARGET RESET** pushbutton on the SEL-551 front panel. This directly unlatches the TRIP bit and closes the dummy breaker immediately (no 100 ms close delay). Use for test reset only.

---

## 6. Anti-Pumping Behaviour

The dummy breaker will **not** accept a close command (IN2 or MAN CLOSE) while a trip command (IN1 or MAN TRIP) is simultaneously asserted. This mirrors real breaker anti-pumping protection.

**Close is blocked when IN1 or LB1 is asserted.** If both close and trip commands are active with the breaker open, the display shows:

```
┌────────────────┐
│ CB OPEN        │
│ ANTI PUMP      │
└────────────────┘
```

**Practical implication:** If the relay-under-test holds its trip output latched after a fault, reset the relay-under-test before attempting to close the dummy breaker.

---

## 7. Front-Panel LCD Display

The display shows a **single static 2-line screen** — it does not rotate between multiple screens.

- **Top row** — always shows the current breaker state:
  - `CB OPEN` when the breaker is open (TRIP latch set)
  - `CB CLOSED` when the breaker is closed (TRIP latch reset)

- **Bottom row** — shows one of two states:

| Bottom row text | Condition |
|-----------------|-----------|
| `ANTI PUMP` | Trip and close commands both asserted with breaker open |
| `OP/CL DLY SV5PU` | All other states |

---

## 8. Indicator Lamps and Front-Panel LEDs

### Built-in Front-Panel LEDs

| LED | Condition | Meaning |
|-----|-----------|---------|
| TRIP (red) | On | Breaker is OPEN (TRIP latch set) |
| TRIP (red) | Off | Breaker is CLOSED |
| ENABLED (green) | On | Relay is powered and healthy |

### External Indicator Lamps (Output Contacts)

Wire lamps to the output contacts for bench-level status at a glance:

| Output | Colour | Label | ON when |
|--------|--------|-------|---------|
| OUT1 | Green | CB CLOSED | Breaker is closed (normal service) |
| OUT2 | Red | CB OPEN | Breaker is open |
| OUT3 | Amber | SPRING CHG | 15 s spring charge block active after CO operation |
| OUT4 | Amber | OPERATING | Opening or closing delay in progress |

---

## 9. Adjusting the Time Delay

A single setting (`SV5PU`) controls both the opening and closing delay. They are always equal. All timers are in **cycles** (1/8-cycle resolution). At 50 Hz: 1 cycle = 20 ms.

| Desired delay | Setting value |
|---------------|---------------|
| 50 ms | 2.500 cycles |
| 83 ms | 4.125 cycles |
| 100 ms | 5.000 cycles |
| 150 ms | 7.500 cycles |
| 200 ms | 10.000 cycles |

### Method A — Via QuickSet or Serial Terminal

1. Change `SV5PU` in SET_1.TXT to the desired cycle count.
2. Send updated settings to the relay (groups 1 and L).

### Method B — Via Relay Front-Panel Menu

1. Press **SET** to enter the settings menu.
2. Press the right-arrow key to scroll through settings until **SV5PU** appears.
3. Press **SELECT** — the current value is shown with the cursor active.
4. Use arrow keys to change the value.
5. Press **SELECT** to confirm. Enter the access-level 2 password if prompted.
6. Verify the new value is shown.

No display label update is required — the bottom row always shows `OP/CL DLY SV5PU`.

---

## 10. Front-Panel Local Controls

V0.6 adds two front-panel soft switches: **MAN TRIP (LB1)** and **MAN CLOSE (LB2)**. These allow the breaker to be operated from the relay front panel without external wiring.

### Operating the Local Controls

1. Press the **CNTRL** pushbutton on the relay front panel. Display shows:
   ```
   Press CNTRL for
   Local Control
   ```

2. Press **CNTRL** again. The first local control switch appears:
   ```
   MAN TRIP
   RETURN
   ```

3. Press **RIGHT ARROW** to scroll between switches (MAN TRIP ↔ MAN CLOSE).

4. With the desired switch shown, press **SELECT**. The operate prompt appears:
   ```
   MAN TRIP
   No  Yes
   ```

5. Press **LEFT ARROW** to move cursor to **Yes**. Press **SELECT** to execute.

Both MAN TRIP and MAN CLOSE are **momentary (type M)** switches — the local bit fires for one processing interval then auto-resets, equivalent to a momentary button press.

| Switch | Equivalent to | Notes |
|--------|--------------|-------|
| MAN TRIP (LB1) | Asserting IN1 | Starts 100 ms opening timer |
| MAN CLOSE (LB2) | Asserting IN2 | Blocked if 0.3 s dead time not expired, or spring charge active |

---

## 11. Spring Charge Simulation (O-0.3s-CO-15s-CO)

V0.6 simulates the standard HV circuit breaker operating sequence:

```
O — 0.3 s — CO — 15 s — CO
```

| Symbol | Meaning |
|--------|---------|
| **O** | Open (trip). Breaker opens. |
| **0.3 s** | Dead time — arc de-ionisation, minimum wait before first reclose. |
| **CO** | Close then immediate Open. Reclose attempt; if fault persists, the RUT re-trips. |
| **15 s** | Spring recharge time — motor must recharge spring before next close. |
| **CO** | Second reclose attempt. |

### How It Works

| Timer | Setting | Function |
|-------|---------|---------|
| `SV6PU` | 15 cycles (0.3 s) | Dead-time timer. Starts when TRIP=1. Close blocked until SV6T=1. |
| `SV7DO` | 50 cycles (1 s) | Reclose memory window. SV7T=1 for 1 s after each close event. |
| `SV8DO` | 750 cycles (15 s) | Spring charge block. SV8T=1 for 15 s after a CO operation. OUT3 energised. |

A CO event is detected when the breaker re-trips within 1 s of a close (SV7T=1 and TRIP returns to 1).

### Sequence Summary

1. **Fault → Trip:** IN1 asserts → 100 ms → breaker opens. SV6 starts counting 0.3 s.
2. **Dead time expires:** SV6T=1 at 0.3 s. First close now permitted.
3. **First reclose (CO):** Assert IN2 / MAN CLOSE → 100 ms → breaker closes. If fault present, RUT immediately re-trips → IN1 asserts again → breaker opens. SV8T=1 for 15 s. OUT3 (SPRING CHG lamp) illuminates.
4. **Spring charge period:** OUT3 ON. Close blocked. Wait 15 s.
5. **Second reclose:** OUT3 extinguishes. SV6T=1 already. Assert IN2 / MAN CLOSE → 100 ms → breaker closes.

### Practical Note

For tests that do not exercise autoreclosing, this behaviour is transparent. The 0.3 s dead time and 15 s spring charge only activate if a CO event occurs. A simple trip → clear fault → close sequence bypasses both timers.

---

## 12. Retrieving Event Records

The SEL-551 stores a Sequential Events Recorder (SER) log capturing all transitions of:

| Signal | What it captures |
|--------|------------------|
| `TRIP` | Breaker state changes (open / close) |
| `IN1` | External trip commands received |
| `IN2` | External close commands received |
| `LB1` | Manual trip (front panel) |
| `LB2` | Manual close (front panel) |
| `SV5T` | Operating timer — 100 ms after trip or close command |
| `SV6T` | Dead time timer — 0.3 s after breaker opened |
| `SV8T` | Spring charge timer — 15 s block after CO operation |

To retrieve via serial terminal:
```
SER       — print Sequential Events Recorder log
EVE       — list stored event reports
EVE n     — print event report number n
```

SER timestamps allow post-test verification of actual timing behaviour.

---

## 13. Limitations and Notes

1. **No current/voltage inputs required.** Do not connect CTs to this relay while configured as a dummy breaker. All overcurrent protection elements are disabled.

2. **Output contact hardware delay.** Output relay contacts have a hardware operate time of approximately 8 ms in addition to the programmed timer delay. Total operating time ≈ 108 ms. This is consistent with real breaker behaviour and measurable via the SER.

3. **Minimum trip duration (TDURD = 3 cycles / 60 ms).** The TRIP latch will not unlatch less than 60 ms after it sets. In practice this never constrains close timing since the close timer adds 100 ms.

4. **Power loss behaviour.** If control power is lost, all output contacts de-energise. OUT1 (52A) opens — the relay-under-test sees the breaker as OPEN. This is fail-open / fail-safe behaviour.

5. **IN1/IN2 input debounce.** The SEL-551 optoisolated inputs have a built-in 0.25-cycle debounce filter. Transients shorter than 5 ms will not be seen by the relay.

6. **Opening and closing delays are equal.** A single timer (`SV5PU`) governs both. The dead-time (0.3 s) and spring-charge (15 s) timers are separate and independent. If independent open and close delays are required, refer to the design plan.

7. **No lockout after third trip.** V0.6 does not implement lockout — after the second CO the spring charge cycle repeats. If lockout is required, refer to the design plan enhancement for 79-element-based lockout.
