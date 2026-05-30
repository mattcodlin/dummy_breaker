# SEL-551 Dummy Circuit Breaker — Design Plan

**Current settings version:** DUMMY_CB_V0.6

---

## Purpose

Configure an SEL-551 relay to simulate a circuit breaker with realistic opening and closing time delays, front-panel local controls, and HV breaker operating sequence simulation (O-0.3s-CO-15s-CO). The dummy breaker is used during protection relay testing to make the relay-under-test behave as if it is connected to a real circuit breaker.

---

## Design Overview

### Core Concept

The SEL-551 TRIP latch is repurposed as the breaker state register:

- **TRIP = 0** → Breaker is CLOSED → OUT1 energised (contact closed)
- **TRIP = 1** → Breaker is OPEN → OUT1 de-energised (contact open)

On power-up the TRIP latch initialises to 0 — the breaker starts CLOSED.

### Input/Output Assignment

| Terminal | Function | Description |
|----------|----------|-------------|
| IN1 | Trip command | From relay-under-test trip output and/or test set |
| IN2 | Close command | From relay-under-test close output, test set, or push button |
| LB1 | Manual trip | Front-panel CNTRL button (MAN TRIP — momentary) |
| LB2 | Manual close | Front-panel CNTRL button (MAN CLOSE — momentary) |
| OUT1 | 52A contact (N/O) + Green lamp | CLOSED when breaker CLOSED; OPEN when breaker OPEN |
| OUT2 | 52B contact (N/O) + Red lamp | CLOSED when breaker OPEN; OPEN when breaker CLOSED |
| OUT3 | Spring charge indicator — Amber lamp | Energised for 15 s after CO operation (close blocked) |
| OUT4 | Operating indicator — Amber lamp | Energised during opening or closing delay |

---

## SELOGIC Logic Design

### Shared Trip/Close Timer (SV5)

A single timer handles both the opening and closing delays:

```
SV5   = IN1 + LB1 + (IN2 + LB2) * TRIP * SV6T * !SV8T
SV5PU = 5 cycles (100 ms)
SV5DO = 0 cycles
```

The close path `(IN2 + LB2) * TRIP * SV6T * !SV8T` is gated by:
- `TRIP=1` — breaker must be open to close
- `SV6T=1` — 0.3 s dead time must have expired
- `!SV8T` — spring charge block must not be active

### Trip Latch

```
TR    = SV5T * (IN1 + LB1)     ; Latch TRIP when trip command caused the timer to fire
ULTR  = SV5T * !IN1 * !LB1     ; Unlatch TRIP when close command caused the timer to fire
TDURD = 3 cycles (60 ms)        ; Minimum trip latch hold time
52A   = !TRIP                   ; Internal breaker status feedback
```

TRIP unlatches only when: TDURD expired, TR=0 (no trip command), and ULTR=1 (timer fired on close side).

### State Indicator Variables (SV1–SV3, untimed)

```
SV1 = (IN1 + LB1) * !TRIP              ; Opening in progress (drives OUT4)
SV2 = (IN2 + LB2) * TRIP * !IN1 * !LB1 ; Closing in progress (drives OUT4)
SV3 = (IN1 + LB1) * (IN2 + LB2) * TRIP ; Anti-pump active (drives DP2)
```

### Dead-Time Timer (SV6)

```
SV6   = TRIP     ; Starts timing when breaker opens
SV6PU = 15 cycles (0.3 s at 50 Hz)
SV6DO = 0 cycles
```

`SV6T=1` after the breaker has been open for 0.3 s. This gates the close path in SV5.

### Reclose Memory Window (SV7)

```
SV7   = SV5T * !IN1 * !LB1   ; Same condition as ULTR — fires when close timer completes
SV7PU = 0 cycles
SV7DO = 50 cycles (1 s at 50 Hz)
```

`SV7T=1` for 1 s after the breaker closes. Used to detect a CO event — if TRIP reasserts while SV7T=1, a reclose attempt followed by re-trip has occurred. (Note: `\TRIP` falling-edge syntax is avoided because edge detection on the TRIP latch is not accepted by the relay.)

### Spring Charge Block (SV8)

```
SV8   = SV5T * (IN1 + LB1) * SV7T   ; CO event: trip timer fires with trip cmd while SV7T active
SV8PU = 0 cycles
SV8DO = 750 cycles (15 s at 50 Hz)
```

`SV8T=1` for 15 s after a CO event. Blocks the close path in SV5. `OUT3 = SV8T`.

**Why not `SV7T * TRIP`:** SELOGIC evaluates SV1–SV14 before updating the TRIP latch. On the scan a close completes, SV7T rises (PU=0) while TRIP is still at its previous-scan value of 1. `SV7T * TRIP` therefore evaluates to 1 on every close — triggering the spring charge block even when there is no CO event. Using `SV5T * (IN1 + LB1) * SV7T` instead means SV8 only fires when a **trip command** caused the timer to fire while a close had recently occurred (i.e., a true CO sequence).

### Output Equations

```
OUT1 = !TRIP       ; 52A: energised when breaker CLOSED (green lamp)
OUT2 =  TRIP       ; 52B: energised when breaker OPEN (red lamp)
OUT3 = SV8T        ; Spring charging: energised for 15 s after CO (amber lamp)
OUT4 = SV1 + SV2   ; Operating: energised during opening or closing delay (amber lamp)
```

### Display Points

| DP | Equation | `_1` text | `_0` text |
|----|----------|-----------|-----------|
| DP1 | `TRIP` | `CB OPEN` | `CB CLOSED` |
| DP2 | `SV3` | `ANTI PUMP` | `OP/CL DLY SV5PU` |
| DP3–DP8 | `0` | — | — |

### Local Control Configuration (SET_T.TXT)

| Field | Value | Meaning |
|-------|-------|---------|
| NLB1 | `MAN TRIP` | Display name |
| CLB1 | `RETURN` | Non-operated position label |
| SLB1 | `TRIP` | Operated position label |
| PLB1 | `M` | Momentary type |
| NLB2 | `MAN CLOSE` | Display name |
| CLB2 | `RETURN` | Non-operated position label |
| SLB2 | `CLOSE` | Operated position label |
| PLB2 | `M` | Momentary type |

---

## Settings Files

| File | Key contents |
|------|-------------|
| SET_1.TXT | All protection OFF; SV5PU=5, SV6PU=15, SV7DO=50, SV8DO=750; TDURD=3 |
| SET_L.TXT | All SELOGIC equations as above |
| SET_T.TXT | Display labels and LB1/LB2 local control configuration |
| SET_R.TXT | SER: TRIP IN1 IN2 LB1 LB2 SV5T SV6T SV8T |
| SET_P1.TXT | Serial port — 19200 baud, 8N1 |

---

## Timing Summary

| Event | Signal path | Delay |
|-------|-------------|-------|
| IN1/LB1 asserts → breaker opens | IN1+LB1 → SV5 → SV5T → TR → TRIP=1 → OUT1=0 | 100 ms |
| IN2/LB2 asserts → breaker closes | IN2+LB2 → SV5 (TRIP=1, SV6T=1, !SV8T) → SV5T → ULTR → TRIP=0 → OUT1=1 | 100 ms |
| Dead time | TRIP=1 → SV6 → SV6T=1 | 0.3 s |
| CO detection window | SV5T * !IN1 * !LB1 → SV7 → SV7T=1 for 1 s | 1 s |
| Spring charge block | SV5T * (IN1+LB1) * SV7T → SV8 → SV8T=1 | 15 s |
| Output contact hardware | Relay coil/contact mechanism | ~8 ms additional |

---

## Timer Settings (cycles at 50 Hz)

| Setting | Value | Time |
|---------|-------|------|
| SV5PU | 5.000 | 100 ms — opening/closing delay |
| SV6PU | 15.000 | 300 ms (0.3 s) — dead time |
| SV7PU | 0.000 | — |
| SV7DO | 50.000 | 1 s — reclose memory window |
| SV8PU | 0.000 | — |
| SV8DO | 750.000 | 15 s — spring charge block |
| TDURD | 3.000 | 60 ms — minimum trip latch hold |

---

## How to Adjust Timing

To change the opening/closing delay: change `SV5PU` in SET_1.TXT.  
To change dead time: change `SV6PU` (default 15 = 0.3 s).  
To change spring charge duration: change `SV8DO` (default 750 = 15 s).

| Desired open/close delay | SV5PU value |
|--------------------------|-------------|
| 50 ms | 2.500 cycles |
| 83 ms | 4.125 cycles |
| 100 ms | 5.000 cycles |
| 150 ms | 7.500 cycles |
| 200 ms | 10.000 cycles |

---

## Suggested Enhancements

### 1. Lockout After N Trips
After N consecutive CO operations without a successful sustained close, lock out the breaker and require a manual front-panel reset. Partially achievable using the 79-element 79LO bit or an SV counter chain.

### 2. Multi-Shot Autorecloser Simulation
Configure the SEL-551 built-in 4-shot autorecloser (79 element) for automatic reclosing with proper dead times, allowing testing of relay-under-test autorecloser supervision without modifying dummy breaker SELOGIC.

### 3. SEL-551C for Additional Inputs
The SEL-551C has 6 inputs vs 2. A third input could be used for sync-check or close-permissive, with further inputs for lamp test or lockout override.

### 4. Independent Open and Close Delays
Restore a separate close timer (SV9) to allow different open and close delay settings.

### 5. Adjustable Spring Charge Period
Expose `SV8DO` as a front-panel-accessible setting by referencing it in the display label. Currently requires a settings file change to adjust.
