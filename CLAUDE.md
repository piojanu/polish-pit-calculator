# CLAUDE.md

Handoff notes for continuing a Polish-tax compliance review of this repo.
Scope of the prior session: confirm the tool's output is correct and usable
for filing PIT-38 for tax year 2025 (deadline 30 April 2026), with particular
focus on a Charles Schwab Employee Sponsored ESPP flow.

## Repo in one paragraph

`polish-pit-calculator` (v0.1.0) aggregates broker/interest/manual inputs into
per-year Polish PIT summaries. Core model is `polish_pit_calculator/config.py`
(`TaxRecord`, `TaxReport`). Reporters live in
`polish_pit_calculator/tax_reporters/` (Schwab EAC JSON, IBKR Flex API,
Coinbase CSV, Revolut Interest CSV, manual Trade / Crypto / Employment).
NBP FX cache in `caches.py` uses Table A mid + `.shift()` to map a trade date
to the previous business-day rate (correct per art. 11a PIT, with one edge
case noted below). Output is a pandas dataframe keyed by year, labelled with
PIT form coordinates.

## PIT-38 for tax year 2025 — the mapping that matters

Form version: PIT-38 v17+ (2025). Key positions confirmed against the
Ministry brochure and multiple guides:

- Section C (odpłatne zbycie papierów wartościowych):
  - poz. 20 / 21: revenue / cost — **domestic** (from PIT-8C part F)
  - poz. 22 / 23: revenue / cost — **foreign** (PIT-8C part G, foreign brokers)
  - poz. 24 / 25: sum revenue / sum cost
  - poz. 26 dochód, poz. 27 strata
- Section D (obliczenie art. 30b):
  - poz. 28 straty z lat ubiegłych (cap: 50% per year for pre-2019 losses;
    2019+ losses may be offset up to 5 M PLN per year)
  - poz. 29 podstawa, poz. 30 stawka 19 %, poz. 31 podatek
- Section E (waluty wirtualne):
  - poz. 34 przychód, poz. 35 koszty roku, poz. 36 koszty z lat ubiegłych
  - No "tax loss" — excess cost carries to future years as cost.
- Section G (zryczałtowany art. 30a): foreign dividends / interest at 19 %,
  credit for foreign WHT capped at Polish 19 % (practically capped at treaty
  rate, typically 15 % for US dividends).

Required attachments for any foreign income:
- **PIT/ZG** (v8) — one per country, attached to PIT-38.
- **DSF-1** — only if solidarity tax (4 % above 1 M PLN income) applies.
- **PIT/O** — for donation relief, etc.

PIT/ZG for PIT-38 stock gains (section C.3 of PIT/ZG):
- poz. 9 country name, poz. 10 country code (`US` for USA)
- poz. 32 dochód (art. 30b) — equals the foreign portion of PIT-38 poz. 26
- poz. 33 podatek zapłacony za granicą — for US capital gains this is 0
- PIT/ZG is required even when no foreign tax was withheld. Dividends under
  art. 30a don't need PIT/ZG (they just go to PIT-38 section G).

## What this tool gets right

- Trade Revenue (`C22`), Trade Cost (`C23`) — **already fixed in this
  session**; previous labels `C20`/`C21` were wrong for foreign brokers.
- `D28` trade-loss carry, `E34/E35/E36` crypto mapping.
- 19 % rate on trade, crypto, interest/dividends; 4 % solidarity above 1 M PLN.
- Crypto excess as carryforward-cost (not tax loss) — matches art. 22 ust. 16.
- NBP Table A, previous-business-day rate via `.shift()` — correct except:
  - Edge bug: if the transaction lands on a non-business day (e.g. 26 Dec),
    the tool returns the rate from *two* NBP business days earlier instead of
    one. Impact is usually negligible (rate drift over ~1 day).

## What still needs manual adjustment before filing

1. **PIT/ZG is mandatory for any foreign broker income** — the tool does not
   emit it. Create one per source country. For Schwab (US):
   - poz. 32 = PLN dochód from sale of ESPP/RSU shares (equals PIT-38 foreign-
     sourced portion of poz. 26)
   - poz. 33 = 0 PLN (US doesn't tax capital gains of non-residents)
2. **IBKR reporter lumps dividends + interest into `foreign_interest`** —
   same 19 % rate, but foreign WHT credit caps differ (treaty 15 % for US
   dividends). If relying on `foreign_interest_withholding_tax`, check you
   aren't over-crediting.
3. **Revolut Interest reporter flags income as `domestic_interest`** — wrong,
   Revolut Bank UAB is Lithuanian; it must be declared on PIT-38 section G
   (foreign), and PIT/ZG for LT is required.
4. **Loss-carryforward limits** — tool subtracts `trade_loss_from_previous_years`
   in full; user must clamp to 50 % (pre-2019) / 5 M PLN (2019+) themselves.
5. **DSF-1 C19 deductions** — tool includes `employment_profit_deduction`
   (6 % donation cap) in `total_profit_deductions`. Art. 30h ust. 2 permits
   only ZUS to reduce the solidarity base. Donations do not.
6. **Schwab RSU cost basis** — `sale cost = PurchasePrice`, which is 0 for
   RSU lots. If the Polish employer taxed the vest FMV as employment income,
   the cost basis at sale is that FMV, not 0. (RSU vest-vs-sale taxation is
   itself disputed in PL; many advisers defer to sale.)
7. **Domestic-Interest row maps to `PIT-38/G44`** — wrong on two grounds.
   Polish bank interest is withheld at source and never appears on PIT-38;
   also G44 is not the current-form position for foreign art. 30a. Ignore
   this row unless you know what it's for.
8. **PIT-38 section G positions on the tool** (`G44/G45/G46`) — uncertain
   against the v17+ form. Re-verify live before filing.
9. **Schwab "stock-split" heuristic** — `schwab.py` has an auto split detector
   that false-positived on the user's 2025 NVDA data (doubled per-share
   PurchasePrice on the ESPP 02/28 lot and halved its quantity). Totals were
   preserved (self-consistent on buy + matching sell), so PLN output was
   unaffected. Worth hardening if you touch that file.

## Reproducing the run in a new session (with network)

```bash
uv sync --group dev
# point the FX cache to a writable dir (or accept ~/.cache)
export POLISH_PIT_CALCULATOR_CACHE_DIR=/tmp/pit_cache
uv run pit-pl
# menu → Register tax reporter → Charles Schwab Employee Sponsored
# → point at your schwab.json (drop identifying fields first, see below)
# → Prepare tax report → Show tax report
```

Recommended JSON sanitize before pasting anywhere outside your machine:

```bash
jq 'del(.AccountNumber, .AccountName, .ClientName, .SSN, .EmployeeID, .EmployerID)' \
  schwab.json > schwab.sanitized.json
```

The tool fetches NBP rates on first run; in this offline sandbox we seeded
`/tmp/pit_cache/2025.csv` from the NBP archive manually. With network, that
step is automatic.

## Outstanding work (if continuing)

- Verify PIT-38 section G field numbers on current v17+ form and update
  `config.py` labels.
- Decide whether to drop or rename the `Domestic Interest Tax` row.
- Fix the IBKR reporter to split dividends vs interest, and apply the
  treaty-rate WHT cap.
- Reclassify Revolut Interest as foreign.
- Enforce the 50 % / 5 M PLN loss-carry cap in `TaxRecord.trade_profit`.
- Remove donations from `total_profit_deductions` (DSF-1 C19 base).
- Support emitting RSU vest values as employment income if the user opts in.
- Harden or gate the Schwab split-detection heuristic.

## Source references

- Broszura PIT-38 za 2025 r. (podatki.gov.pl)
- PIT/ZG instructions (PITax / pit.pl / stockbroker.pl)
- art. 30a, 30b, 30h ustawy o PIT
- Polska–USA umowa o unikaniu podwójnego opodatkowania (15 % dividend cap)
