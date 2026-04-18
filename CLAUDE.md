# CLAUDE.md

Handoff notes for continuing Polish-tax compliance work on this repo.

Scope covered so far: PIT-38 filing for tax year 2025 (deadline 30 April 2026),
with the Charles Schwab Employee Sponsored ESPP/RSU flow as the reference
case. Every TODO in "Outstanding work" below is cross-referenced to the
specific file and function that needs touching.

## Repo in one paragraph

`polish-pit-calculator` aggregates broker/interest/manual inputs into
per-year Polish PIT summaries. Core model is `polish_pit_calculator/config.py`
(`TaxRecord`, `TaxReport`). Reporters live in
`polish_pit_calculator/tax_reporters/` (Schwab EAC JSON, IBKR Flex API,
Coinbase CSV, Revolut Interest CSV, manual Trade / Crypto / Employment).
NBP FX cache in `caches.py` uses Table A mid + `.shift()` to map a trade date
to the previous business-day rate (correct per art. 11a PIT, with one edge
case noted below). Output is a pandas dataframe keyed by year, labelled with
PIT form coordinates. 100 % test coverage is enforced (`tool.coverage.report`
`fail_under = 100`), and `pit-pl` is the CLI entrypoint.

## PIT-38 for tax year 2025 — verified v18 layout

Form version: **PIT-38 v18** (the 2025 brochure
`broszura-do-pit-38-za-2025-r.pdf` on podatki.gov.pl, published 2025-12-08).
Positions below are read directly from that brochure:

- Section C (odpłatne zbycie papierów wartościowych, art. 30b):
  - poz. 20 / 21: revenue / cost — **domestic** (PIT-8C part F, poz. 35/36)
  - poz. 22 / 23: revenue / cost — **foreign** (PIT-8C part G + own records)
  - poz. 24 / 25: *wiersz 3 — amounts included in PIT-8C that should be
    excluded* (e.g. items under a different regime). Not a "sum" row.
  - poz. 26 / 27: sum revenue / sum cost = (20 + 22 − 24) / (21 + 23 − 25)
  - poz. 28 dochód, poz. 29 strata
- Section D (obliczenie podatku — art. 30b):
  - poz. 30 straty z lat ubiegłych (cap: 50 % per year for pre-2019 losses;
    2019+ losses may be offset up to 5 M PLN per year — not enforced by the
    tool, see TODO)
  - poz. 31 podstawa opodatkowania (26 − 30, rounded to złote)
  - poz. 32 stawka 19 %
  - poz. 33 podatek (19 % × poz. 31, rounded)
  - poz. 34 podatek zapłacony za granicą (art. 30b ust. 5a/5b, capped at 19 %)
  - poz. 35 podatek należny
- Section E (waluty wirtualne, art. 30b ust. 1a):
  - poz. 36 przychód, poz. 37 koszty roku, poz. 38 koszty z lat ubiegłych
  - poz. 39 dochód, poz. 40 nadwyżka kosztów (carryforward-cost, not a tax loss)
  - poz. 41 podstawa, poz. 42 stawka 19 %, poz. 43 podatek, poz. 44 podatek
    zapłacony za granicą (art. 30b ust. 5e/5f), poz. 45 podatek należny
- Section G (zryczałtowany art. 30a, zagraniczny):
  - poz. 46 suma zryczałtowanego podatku dochodowego
  - poz. 47 podatek obliczony od przychodów art. 30a ust. 1 pkt 1–5 uzyskanych
    za granicą (foreign dividends/interest at 19 %)
  - poz. 48 podatek zapłacony za granicą (credit, capped at Polish 19 %;
    practically capped at treaty rate — 15 % for US dividends)
  - poz. 49 różnica (47 − 48)

There is **no "domestic interest" field on PIT-38**. Polish bank interest is
withheld at source under art. 30a and is final; it never appears on the form.

Required attachments:
- **PIT/ZG** (v8) — one per country, attached to PIT-38. Required even when
  no foreign tax was withheld. Section C.3 of PIT/ZG:
  - poz. 9 country name (e.g. "Stany Zjednoczone Ameryki")
  - poz. 10 country code (`US` for USA)
  - poz. 29 dochód (art. 30b) — equals the foreign portion of PIT-38 poz. 28
  - poz. 30 podatek zapłacony za granicą — 0 PLN for US capital gains
    (confirmed on the live e-PIT v8 form during the 2025 filing)
- **DSF-1** — only if solidarity tax (4 % above 1 M PLN income) applies.
- **PIT/O** — for donation relief, etc.

Dividends under art. 30a don't need PIT/ZG — they go straight into PIT-38
section G.

## Why dividends are "zryczałtowany" (and capital gains aren't)

`Zryczałtowany podatek` is a distinct tax regime, not just a word. When you
see it on a PIT-38 field label (notably poz. 46 and 47), it signals four
properties that apply together:

1. **Fixed rate** — 19 % flat for art. 30a ust. 1 pkt 1–5 (interest, bond
   discount, dividends, capital-fund distributions). No progressive scale.
2. **Applied to *przychód* (gross), not dochód** — you cannot deduct
   brokerage fees, FX costs, or any acquisition cost from the dividend
   base. This is why section G has no "koszty" field. Contrast with
   section C (art. 30b): poz. 22 is przychód, poz. 23 subtracts costs,
   poz. 28 is the dochód that gets taxed.
3. **Settled in isolation** — never pooled with skala (art. 27) or
   liniowy (art. 30c) income. Dividend tax does not add to or draw from
   the capital-gains calculation. Losses don't cross the boundary either.
4. **Usually collected by płatnik at source** — hence the Część I poz. 65
   declare-only line (art. 45 ust. 3c) when a Polish brokerage or bank
   already withheld. When no płatnik is involved (foreign broker), the
   taxpayer self-settles via section G poz. 47–49, using the art. 30a
   ust. 9 credit mechanism for any foreign WHT (capped at Polish 19 %;
   practically capped at the treaty rate — 15 % for US).

The practical upshot for a Schwab-US filer (no Polish płatnik):

- Poz. 46 — leave 0 (nothing withheld in Poland).
- Poz. 47 — 19 % × Σ(gross USD dividend × NBP_{T-1}), rounded *up* to grosz
  per art. 63 § 1 ordynacji podatkowej (Polish rounding rule for kwoty
  w groszach in art. 30a ust. 1 pkt 1–3).
- Poz. 48 — Σ(USD WHT × NBP_{T-1}), capped at poz. 47.
- Poz. 49 — poz. 47 − poz. 48, rounded to full złote.

e-PIT (`epit.podatki.gov.pl`) renders these on the same screen as one
card titled *"Zryczałtowany podatek od przychodów (dochodów), w tym
uzyskanych poza granicami Polski"*; poz. 46 is pre-filled/locked (what
Polish płatnik withheld), poz. 47 and 48 are free-form, poz. 49 is
computed.

## Label mapping in the tool (current state)

`TaxRecord.get_name_to_pit_label_mapping()` in `config.py` uses the v18
coordinates. The row→poz. mapping is:

| Row                                   | Position                     |
|---------------------------------------|------------------------------|
| Trade Revenue                         | PIT-38/C22                   |
| Trade Cost                            | PIT-38/C23                   |
| Trade Loss from Previous Years        | PIT-38/D30                   |
| Trade Loss (carryforward)             | PIT-38/D30 — Next Year       |
| Crypto Revenue                        | PIT-38/E36                   |
| Crypto Cost                           | PIT-38/E37                   |
| Crypto Cost Excess from Previous Yrs  | PIT-38/E38                   |
| Crypto Cost Excess (carryforward)     | PIT-38/E38 — Next Year       |
| Domestic Interest Tax                 | *(blank — not on PIT-38)*    |
| Foreign Interest Tax                  | PIT-38/G47                   |
| Foreign Interest Withholding Tax      | PIT-38/G48                   |
| Employment Profit Deduction           | PIT/O/B11 → PIT-37/F124      |
| Total Profit                          | DSF-1/C18 (if solidarity > 0)|
| Total Profit Deductions               | DSF-1/C19 (if solidarity > 0)|

## What still needs manual adjustment before filing

The tool computes PLN amounts correctly, but several things must still be
done by hand:

1. **PIT/ZG is not emitted.** Create one per source country. For a Schwab-US
   filer: poz. 29 = the foreign portion of PIT-38 poz. 28; poz. 30 = 0 PLN.
2. **Loss-carryforward limits** — `TaxRecord.trade_profit` subtracts
   `trade_loss_from_previous_years` in full. User must clamp to 50 %
   (pre-2019 losses) / 5 M PLN (2019+ losses) per year themselves.
3. **Schwab RSU cost basis** — `schwab.py` uses `PurchasePrice` (0 for RSU)
   as the cost basis at sale. If the Polish employer taxed vest FMV as
   employment income, cost basis at sale is that FMV. RSU vest-vs-sale
   taxation is disputed in PL; many advisers defer to sale.
4. **DSF-1 C19 deductions** — `TaxRecord.total_profit_deductions` includes
   `employment_profit_deduction` (6 % donation cap). Art. 30h ust. 2 permits
   only ZUS to reduce the solidarity base. Donations do not. Subtract it
   manually if solidarity applies.
5. **IBKR reporter lumps dividends + interest into `foreign_interest`** —
   same 19 % rate, but WHT credit caps differ (treaty 15 % for US dividends).
   If relying on `foreign_interest_withholding_tax`, check for over-crediting.
6. **Revolut Interest reporter classifies income as `domestic_interest`**.
   Revolut Bank UAB is Lithuanian — it must be declared on PIT-38 section G
   (foreign) and PIT/ZG for LT is required.
7. **NBP edge case**: if the transaction lands on a non-business day (e.g.
   26 Dec), `.shift()` returns the rate from two NBP business days earlier
   instead of one. Impact is usually a fraction of a PLN. Verify critical
   dates by hand.

## Outstanding code work (roughly in priority order)

- **PIT/ZG emission.** Add a per-country section to `TaxReport` /
  `to_dataframe()` that surfaces foreign-sourced trade-profit PLN and
  foreign-paid tax, grouped by country. Reporters already track the
  source broker — thread country code through `TaxRecord` (new field) or
  carry a parallel structure on `TaxReport`.
- **Revolut reclassification** (`tax_reporters/revolut.py`): move income
  from `domestic_interest` to `foreign_interest` and surface the LT
  country code for PIT/ZG.
- **IBKR split** (`tax_reporters/ibkr.py`): separate dividends from
  interest in the reporter output; apply a treaty WHT cap before adding
  to `foreign_interest_withholding_tax`.
- **Loss-carryforward cap** (`config.py` — `TaxRecord.trade_profit`):
  clamp the carried loss to min(stated, 0.5 × loss) for pre-2019 vintages
  and min(stated, 5e6) for 2019+. Requires tracking vintage on the input.
- **DSF-1 base fix** (`config.py` — `total_profit_deductions`): drop
  `employment_profit_deduction`, keep only `social_security_contributions`.
- **RSU vest income opt-in** (`tax_reporters/schwab.py`): emit
  `VestFairMarketValue × shares × FX(vest_date − 1 biz day)` as
  `employment_revenue` and carry that forward as cost basis at sale.
- **NBP non-business-day fix** (`caches.py`): walk back one NBP business
  day when the trade date itself is not in the table, instead of relying
  on `.shift()` plus dict lookup.

## Reproducing the Schwab run in a new session (network required)

```bash
uv sync --group dev
export POLISH_PIT_CALCULATOR_CACHE_DIR=/tmp/pit_cache   # or accept ~/.cache
uv run pit-pl
# menu → Register tax reporter → Charles Schwab Employee Sponsored
#      → point at your schwab.json (sanitize first, see below)
#      → Prepare tax report → Show tax report
```

Recommended JSON sanitize before sharing:

```bash
jq 'del(.AccountNumber, .AccountName, .ClientName, .SSN,
        .EmployeeID, .EmployerID)' \
  schwab.json > schwab.sanitized.json
```

The tool fetches NBP rates on first run. In an offline sandbox, seed
`$POLISH_PIT_CALCULATOR_CACHE_DIR/<year>.csv` from the NBP archive
(`archiwum_tab_a_<year>.csv`).

## Source references

- Broszura informacyjna do PIT-38 za 2025 r. (podatki.gov.pl, 2025-12-08)
- PIT/ZG (v8) — gov.pl PDF
- art. 30a, 30b, 30h ustawy o PIT (Dz.U. 1991 nr 80 poz. 350 ze zm.)
- Polska–USA umowa o unikaniu podwójnego opodatkowania (15 % dividend cap)
