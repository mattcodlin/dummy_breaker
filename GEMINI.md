# SEL-551 Dummy Circuit Breaker — AI Design & Evolution Log

This document records the AI-assisted design process and development history of the SEL-551 Dummy Circuit Breaker Simulator. The settings were designed and iterated through collaboration between the testing technician and the Gemini AI coding assistant.

---

## 1. The Engineering Challenge

The SEL-551 is a feeder overcurrent relay. The goal was to repurpose it into a **logic-only circuit breaker simulator** to facilitate testing. The simulator needed to:
1. Simulate realistic breaker opening and closing times (100 ms).
2. Respond to external inputs (`IN1` for trip, `IN2` for close) and front-panel soft switches (`LB1`, `LB2`).
3. Reflect correct physical status contacts (52A via `OUT1` and 52B via `OUT2`).
4. Provide appropriate front-panel LCD status feedback.
5. Prevent closing during active trip commands (Anti-pumping).
6. Enforce realistic operational constraints, specifically the HV duty cycle: **Open — 0.3s — Close-Open — 15s — Close-Open**.

---

## 2. Key Logical Breakthroughs

### Latching Breaker State
The SEL-551 does not have an internal breaker control block. The design leverages the relay's built-in **TRIP latch** (`TR`/`ULTR`) to track breaker state:
- `TRIP = 0` (latch reset) → Breaker is **CLOSED**.
- `TRIP = 1` (latch set) → Breaker is **OPEN**.

### Solving the Split-Timer Race Condition (v0.5)
In early versions (v0.0 to v0.4), the open path used timer `SV5` and the close path used timer `SV6`. Because the relay evaluates logic sequentially and the inputs can assert simultaneously during testing, this caused race conditions, leading to unexpected anti-pump lockouts or missed timings. 
**Breakthrough:** In v0.5, we consolidated both delays into a **single operating timer** (`SV5`):
- `SV5 = IN1 + IN2 * TRIP`
- Setting the TRIP latch occurs on timer expiration ONLY if a trip command was active: `TR = SV5T * IN1`.
- Unlatching the TRIP latch occurs ONLY if a close command was active: `ULTR = SV5T * !IN1`.

### Close-Open (CO) Detection and Spring Recharge Simulation (v0.6)
Simulating the spring charge motor recharge time (15 s block) after a Close-Open (CO) operation was difficult due to the lack of transition-edge variables in SELOGIC. 
**Breakthrough:** We established a 1 s reclose memory window (`SV7` timer with drop-out `SV7DO` = 50 cycles). When a close completes, `SV7` is energized for 1 s. If a trip command is received while `SV7T = 1`, it indicates a CO sequence. This triggers the spring charge block `SV8` (`SV8DO` = 750 cycles = 15 s) which gates the close path in `SV5`.

---

## 3. Version Log & Logic Evolution

### Version 0.0: Initial Proof of Concept
Established the core latching concept and independent delays.
- **SV5 (Open Delay):** `IN1` (5 cycles / 100 ms)
- **SV6 (Close Delay):** `IN2 * !IN1` (5 cycles / 100 ms)
- **Latch logic:** `TR = SV5T`, `ULTR = SV6T`
- **Output mappings:** `OUT1 = !TRIP` (52A), `OUT2 = TRIP` (52B), `OUT3 = SV1` (opening), `OUT4 = SV2` (closing)

### Version 0.2: Part Number & LCD Label Updates
- Fixed relay part number formatting in headers.
- Altered front-panel LCD branding text from "DUMMY BREAKER" to "DUMMY CB".

### Version 0.3: Anti-Pumping & Reclose Windows
Introduced basic anti-pumping protection and reclose indicators.
- **SV3 (Anti-pump):** `IN1 * IN2 * TRIP`
- **SV7 (Reclose memory):** `!SV7T` (1 s drop-out)
- **LCD points:** Displayed `ANTI PUMP` and status markers.

### Version 0.4: Dynamic Parameterization
Updated front-panel LCD screens to dynamically display target settings names rather than hardcoded string values.
- Changed display text to show `OPEN DLY SV5PU` and `CLOSE DLY SV6PU`.

### Version 0.5: Consolidated Timer Architecture
Eliminated race conditions by running both trip and close operations through a single physical timer (`SV5`).
- **SV5 (Shared Timer):** `IN1 + IN2 * TRIP`
- **Latch logic:** `TR = SV5T * IN1`, `ULTR = SV5T * !IN1`
- **SV6 (Disabled):** Set to 0.

### Version 0.6: Full Simulation (Duty Cycle + Local Softkeys)
Added the complete O-0.3s-CO-15s-CO sequence simulation and front-panel local push buttons.
- **Local Buttons:** Added `LB1` (`MAN TRIP`) and `LB2` (`MAN CLOSE`).
- **SV5 (Consolidated Shared Timer):** `IN1 + LB1 + (IN2 + LB2) * TRIP * SV6T * !SV8T`
- **SV6 (Dead-time Timer):** `TRIP` (starts timing when breaker opens, 0.3 s delay).
- **SV7 (Reclose Memory):** `SV5T * !IN1 * !LB1` (energized for 1 s after close).
- **SV8 (Spring Charge Block):** `SV5T * (IN1 + LB1) * SV7T` (15 s block triggered by a trip during the reclose window).
- **Output mappings:**
  - `OUT1 = !TRIP` (52A status contact, Green Lamp)
  - `OUT2 = TRIP` (52B status contact, Red Lamp)
  - `OUT3 = SV8T` (Spring Charge Block, Amber Lamp)
  - `OUT4 = SV1 + SV2` (Breaker Operating, Amber Lamp)
- **Latch logic:** `TR = SV5T * (IN1 + LB1)`, `ULTR = SV5T * !IN1 * !LB1`
- **SER Logging:** Configured to record transitions of `TRIP`, `IN1`, `IN2`, `LB1`, `LB2`, `SV5T`, `SV6T`, and `SV8T`.
