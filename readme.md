# FX Position Size Calculator

A Babypips-inspired pip, reward/risk, and position-sizing calculator built with Vue 3 + Vite. Enter TP/Entry/SL and the tool figures out pips, risk capital, pip value per lot, and the position size, while fetching live ask prices for the selected FX pair.

## Quick Start

```bash
npm install
npm run dev
```

Open the dev-server URL, pick a currency pair, click **Refresh Price**, fill in your account inputs, and hit **Calculate Position Size**.

## Build for Production

```bash
npm run build
npm run preview   # optional sanity check of the build
```

## Features
- Fuzzy search for dozens of FX pairs (configurable via `currency_pairs.json`).
- Live ask prices and conversion rates pulled from AwesomeAPI with caching.
- Auto-detected pip precision, pip-value conversion to account currency, and lot/unit sizing.
- Local persistence of the full form + search query for quick scenario tweaking.

Feel free to extend the JSON pair list or styling as you iterate.
