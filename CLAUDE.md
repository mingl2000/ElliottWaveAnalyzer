# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An Elliott Wave scanner for OHLC financial data. The core idea is brute force: generate **a lot** of candidate wave patterns for a chart, then validate each against a rule set (e.g. a 12345 impulse). It is a research prototype — several pieces are explicitly marked "not working atm" in the readme (`WaveCycle`, the full iterative `WaveAnalyzer.next_cycle`).

## Commands

```bash
# Setup (Python 3.9)
pip install -r requirements.txt

# Run an example (MUST run from repo root — see below)
python example_monowave.py            # basic MonoWave concept, play with skip_n
python example_12345_impulsive_wave.py # full scan for 12345 impulses on btc-usd_1d.csv
python example_waveoptions.py         # inspect the WaveOptions combinations

# Download fresh OHLC data from Yahoo Finance into data/
python get_data.py                    # edit ticker/dates inside the script

# Tests (pytest is not in requirements.txt — pip install pytest first)
pytest
pytest tests/test_monowave.py::test_monowave_instance_is_created
```

**Always run scripts from the repo root.** Imports use the `models.` package prefix and data is loaded via hardcoded relative paths like `data\btc-usd_1d.csv` (Windows-style separators in the source). Running from a subdirectory breaks both.

**Plots do not display.** `fig.show()` is commented out in every plotting helper; charts are instead written as PNGs to `./images/` (created on demand, gitignored) via `kaleido`. To see a plot inline, uncomment the `fig.show()` line in the relevant function in [models/helpers.py](models/helpers.py).

## Data format

CSV with columns `Date, Open, High, Low, Close` (no `Adj Close`/`Volume`). Yahoo Finance data must be passed through `convert_yf_data()` in [models/helpers.py](models/helpers.py) to drop/rename columns before saving.

## Architecture

The build-up of abstractions, smallest to largest:

1. **`MonoWave`** ([models/MonoWave.py](models/MonoWave.py)) — the atomic unit: a single directional move (`MonoWaveUp` / `MonoWaveDown`) from a low to a high (or vice versa) where each candle extends the micro-trend, ending when a candle breaks it. The `skip` parameter lets the wave "skip over" N intermediate extrema so smaller counter-moves are absorbed into one larger wave. End-finding delegates to the `@njit`-compiled scanners `hi`/`lo`/`next_hi`/`next_lo` in [models/functions.py](models/functions.py). A `find_end` returning `None` means the wave has no valid end in the data.

2. **`WaveOptions` / `WaveOptionsGenerator`** ([models/WaveOptions.py](models/WaveOptions.py)) — a `WaveOptions` is the tuple of `skip` values, one per MonoWave in a pattern (`[i,j,k,l,m]` for an impulse, `[i,j,k]` for a correction). The `Generator{2,3,5}(up_to=N)` classes enumerate all combinations up to `[N,...,N]` into a **set**, pruning invalid ones (once a skip is 0, all later skips are forced to 0). `options_sorted` orders them small→large so the shortest patterns are tried first. `WaveOptions` implements `__hash__`/`__eq__`/`__lt__` to support the set and the sort.

3. **`WaveAnalyzer`** ([models/WaveAnalyzer.py](models/WaveAnalyzer.py)) — given a DataFrame and a start index, `find_impulsive_wave(idx_start, wave_config)` chains 5 alternating MonoWaves and `find_corrective_wave` chains 3, returning the list of MonoWaves or `False` if any leg has no end / basic geometry fails. This is where a single `WaveOptions` tuple becomes a concrete set of waves.

4. **`WavePattern`** ([models/WavePattern.py](models/WavePattern.py)) — wraps the MonoWave list into a `waves` dict keyed `wave1..waveN` and exposes `check_rule(waverule)`, `dates`/`values`/`labels` (for plotting), and `__eq__`/`__hash__` based on the highs/lows. Equality matters: different `WaveOptions` can produce the identical pattern, so scans dedupe found patterns in a set.

5. **`WaveRule`** ([models/WaveRules.py](models/WaveRules.py)) — the validation layer. Concrete rules (`Impulse`, `Correction`, `TDWave`, `LeadingDiagonal`) subclass `WaveRule` and implement `set_conditions()`, returning a dict of conditions. Each condition is `{'waves': [...], 'function': lambda ..., 'message': ...}`. `WavePattern.check_rule` looks up the named waves, calls the lambda, and the pattern is valid only if **every** condition returns `True`. Add a new rule by subclassing and implementing `set_conditions`.

**The canonical scan loop** (see [example_12345_impulsive_wave.py](example_12345_impulsive_wave.py)): iterate `generator.options_sorted` → `wa.find_impulsive_wave(...)` → build `WavePattern` → `check_rule` against each rule → skip if already in the seen-set → plot. `WaveAnalyzer.next_cycle` attempts to chain an impulse + a correction into a `WaveCycle` but is incomplete.

## Notes / known rough edges

- `WaveOptionsGenerator2.populate` has a bug: `checked = list` (the type, not `list()`) then calls `.append` — it will raise. `Generator3`/`Generator5` are the working ones.
- `WaveAnalyzer.find_td_wave` calls `MonoWaveUp(self.df, ...)`, the old constructor signature — it no longer matches `MonoWave.__init__(lows, highs, dates, idx_start, skip)`. Only the impulsive/corrective finders are current.
- Rule `function` lambdas are named `wave1, wave2...` even inside `Correction`/`TDWave` where they represent A/B/C — the dict keys `wave1..wave3` are positional, not semantic.
