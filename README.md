# Name Screening Engine

A small, explainable sanctions / name-screening engine in Python and SQL, with an evaluation
harness that measures what each design choice actually buys you.

**The question it answers:** *at the same level of risk coverage, how many fewer alerts does
an analyst have to review when screening is done carefully rather than simply?*

> **Synthetic data.** Every customer and watchlist entry in the experiment is generated.
> Results demonstrate the method on a controlled problem; they are not a claim about
> performance on real lists or real customers.

## Result (holdout data, recall target 90%)

Thresholds were chosen on one synthetic dataset (seed 42) and measured on a different one
(seed 7). 2,500 customers, 250 true hits, 1,500 watchlist entries.

| Method | Alerts | False positives | Hits caught |
|---|---|---|---|
| 1. Exact match | 223 | 184 | 15.6% |
| 2. Fuzzy string (raw names) | 1,180 | 952 | 91.2% |
| 3. Fuzzy string (normalised names) | 1,116 | 888 | 91.2% |
| 4. Token alignment | 601 | 385 | 86.4% |
| 5. + rare-token weighting | 571 | 358 | 85.2% |
| 6. Full engine (+ DOB, country, type) | 296 | 82 | 85.6% |

**Read this table with care: the rows do not catch the same share of hits.** Each threshold was set so the
method caught 90% of hits on the development data. On the holdout data the fuzzy baselines held up (91%) but the
token-based methods fell to 85-86%. With 250 hits, a few points is within noise, but the gap is real enough to
state. The fair comparison asks how many alerts each method needs to reach the same recall on the holdout data
(an upper bound on what tuning could achieve, not a number a team could set in advance):

| Alerts needed to catch | 85% of hits | 90% of hits | 95% of hits |
|---|---|---|---|
| Fuzzy string (raw names) | 837 | 1,168 | 1,867 |
| Fuzzy string (normalised names) | 805 | 998 | 1,840 |
| Token alignment + rare-token weighting | 571 | 961 | 1,880 |
| Full engine | 287 | 471 | 1,628 |

* At 85% and 90% coverage the full engine needs roughly half to a third of the alerts of the best fuzzy baseline.
  The biggest single gain comes from secondary data (date of birth, country, entity type); name-matching
  refinements alone help much less.
* **At 95% coverage the advantage nearly disappears** (1,628 against 1,840 alerts, out of 2,500 customers). The
  last hits are heavily damaged names, and catching them is expensive for every method.
* Exact matching still produces 184 false positives, because deliberately created "same name, different
  person" customers match exactly. Name alone cannot resolve them.
* The remaining misses cluster in word drops, short forms and double errors (see the error analysis in the report).

Full report with charts and SQL analysis: [`reports/demo/REPORT.md`](reports/demo/REPORT.md)

## Run on the real OFAC list

The loader reads OFAC's `SDN.CSV` / `ALT.CSV`. It was run on a real `SDN.CSV` downloaded from OFAC with three
made-up customers (`data/sample_customers.csv`); the list loaded and screened without errors. The only candidates
were for a made-up company whose best score was 62.4, against real entries sharing only the word "Trading", so no alert
was raised at the 85 threshold. Each candidate carries an explanation (`examples/sample_output_ofac.csv`, saved from before the fixes, so it still shows the stray "s" and "a" tokens).

That run exposed two gaps, both now fixed: dotted legal suffixes ("S.A.") were split into stray letters, and the
engine did not connect Russian and Belarusian romanisations of the same name.

**Worked example: one real SDN entry (public data), four made-up customers**

| Customer | Result |
|---|---|
| "Siarhei Kastsianok", same date of birth and country | score 96, alert |
| "Kastsianok Siarhei Uladzimiravich", no other data | score 100, alert |
| "Sergei Kostenok" (other romanisation), same date of birth and country | score 88, alert (was: no candidate at all) |
| "Siarhei Kastsianok", different date of birth and country | score 51, no alert |

These are unit-tested (`tests/test_slavic_and_suffixes.py`). The fixes barely move the synthetic experiment,
which contains only a handful of such cases, so the numbers above do not prove them; the unit tests do. Everything here
is a smoke test of loader and scoring, **not an evaluation**: a few made-up customers with no known answer cannot
measure recall or false positives.

## How it works

1. **Normalise**: case, accents, Cyrillic transliteration, punctuation, titles, legal suffixes (including
   dotted ones such as S.A.), spelling variants (Mohammed / Muhammad), and Russian / Belarusian / Ukrainian
   romanisation differences (Siarhei / Sergei, Uladzimir / Vladimir, -au / -ov).
2. **Block**: Soundex buckets keep the candidate set small so screening scales.
3. **Align tokens**: pair name parts one-to-one by similarity; allow initials and word order changes.
4. **Weight**: rare tokens count more than common ones (inverse document frequency).
5. **Score**: harmonic mean of how much of each name is covered, plus a joined-string fallback.
6. **Adjust**: date of birth, country and entity type raise or lower the score. Every adjustment is logged.
7. **Explain**: each alert carries a plain-language reason an analyst can read.

Details and the reasoning behind each choice: [`docs/METHODOLOGY.md`](docs/METHODOLOGY.md)

## Run it

    pip install -r requirements.txt
    python -m pytest -q                       # 45 tests
    python -m screening demo --out reports/demo   # about 30 seconds
    python -m screening screen --watchlist-sdn data/SDN.CSV --customers data/sample_customers.csv

SQL analysis queries live in [`sql/`](sql/); each file states the question it answers.

## Limits (read before trusting any number)

* Synthetic names are less messy than real ones; real performance would differ.
* Ground truth is known by construction, which real screening never has.
* One generator, two seeds. No confidence intervals; differences of a few alerts are noise.
* Soundex favours English-sounding names; non-Latin and Chinese names are handled more weakly.
* Slavic romanisation folding is a hand-written rule set and a small given-name table. It covers common Belarusian,
  Ukrainian and Russian patterns, not Arabic, Chinese or other scripts. Real name variation is far wider.
* Passport and national ID numbers in OFAC remarks are not used; real screening would use identifiers.
* A conflicting date of birth lowers the score, which can suppress a true match when the customer's date is mistyped.
* A screening tool supports a human decision. It does not replace one.
