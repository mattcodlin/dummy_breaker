# SEL-551 Dummy Circuit Breaker Simulator Settings

This repository houses the logic configurations and documentation required to repurpose an **SEL-551 overcurrent protection relay** as a logic-only circuit breaker simulator. This simulator is used to mimic realistic circuit breaker behavior (opening and closing delays, dead-time block, and spring recharge block) during protection scheme testing.

---

## 📘 Quick Documentation Links
- **[Technician Connection & Operation Guide](docs/DUMMY_BREAKER_TECHNICIAN_GUIDE.md)**
  - *Hosted web page: **[DUMMY_BREAKER_TECHNICIAN_GUIDE (GitHub Pages)](https://mattcodlin.github.io/dummy_breaker/DUMMY_BREAKER_TECHNICIAN_GUIDE.html)***
- **[SELOGIC Control & Logic Design Plan](docs/DUMMY_BREAKER_PLAN.md)**
  - *Hosted web page: **[DUMMY_BREAKER_PLAN (GitHub Pages)](https://mattcodlin.github.io/dummy_breaker/DUMMY_BREAKER_PLAN.html)***
- **[SEL-551 Official Instruction Manual (PDF)](docs/551_IM_20250131.pdf)**
- **[AI Design & Settings Evolution Log](GEMINI.md)**

---

## Repository Structure

The repository is organized into settings files and reference documentation:

```
├── README.md               # Repository overview (this file)
├── GEMINI.md               # AI-assisted design process & logic settings evolution log
├── LICENSE                 # MIT License file
├── settings/               # Active SEL-551 settings text files (v0.6)
│   ├── SET_1.TXT           # Global/Element settings (all protection elements disabled, timers)
│   ├── SET_L.TXT           # SELOGIC control equations (state latch, logic logic)
│   ├── SET_T.TXT           # Display point text labels & manual control pushbuttons
│   ├── SET_R.TXT           # Sequential Events Recorder (SER) logging config
│   └── SET_P1.TXT          # Serial port communication parameters (19200 baud, 8N1)
└── docs/                   # User manuals and design guides (also hosts GitHub Pages)
    ├── 551_IM_20250131.pdf               # SEL-551 Instruction Manual
    ├── DUMMY_BREAKER_PLAN.md             # Logic design specifications
    ├── DUMMY_BREAKER_PLAN.html           # Logic design specifications (HTML version for GH Pages)
    ├── DUMMY_BREAKER_TECHNICIAN_GUIDE.md # Technician connection and operation guide
    ├── DUMMY_BREAKER_TECHNICIAN_GUIDE.html # Connection and operation guide (HTML version for GH Pages)
    ├── index.html                        # Automatic redirect to Technician Guide
    └── .nojekyll                         # Bypasses Jekyll build processing
```

---

## Setting Up GitHub Pages

This repository is set up to use GitHub's native Pages branch deployer to serve documentation directly from the `docs/` folder.

To enable it:
1. Navigate to your repository on **GitHub**.
2. Click on **Settings** in the top navigation tab.
3. Select **Pages** from the left sidebar menu.
4. Under **Build and deployment**:
   - Set **Source** to **Deploy from a branch**.
   - Under **Branch**, select `main` and set the folder dropdown to `/docs`.
   - Click **Save**.

Once configured, GitHub will automatically compile the site from the `/docs` folder on every push.
