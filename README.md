# CAST: Cross-Asset State-Space Trading System

Code and data for reproducing the main results of CAST.

---

## Repository Layout

```
CAST/
├── data/raw/
│   ├── NASDAQ.{parquet,json}
│   ├── CSI300.{parquet,json}
│   ├── TPX100.{parquet,json}
│   └── Global30.{parquet,json}
├── src/
│   ├── metrics.py
│   ├── mpc.py
│   ├── backtest.py
│   └── filter/
│       ├── single_ckf.py
│       └── cokf.py
└── experiments/
    └── m30_main_results.py
```

---

## Datasets

Four equity-market panels of 30 daily-close prices each, spanning **January 2005 through April 2025** (5-year calibration window + ~15-year test window):

| Dataset    | Description                                          |
|------------|------------------------------------------------------|
| NASDAQ     | U.S. large-cap, sector-balanced                      |
| CSI300     | Chinese A-share large-cap                            |
| TPX100     | Japanese blue-chips                                  |
| Global30   | Cross-currency basket (multi-currency stress test)   |

Each `.parquet` is the price panel; each `.json` is the metadata (description, selection method, ticker list, shape).

Global30 prices are normalized by their first-day price before backtesting (cross-currency); the other three datasets use raw prices.

---

## Setup

```bash
pip install -r requirements.txt
```

Requires Python ≥ 3.10. (No GPU required; the entire pipeline runs on commodity CPU.)

---

## Reproducing the Main Results

```bash
python3 -u experiments/m30_main_results.py
```

Settings (paper-locked):

| Parameter             | Value                              |
|-----------------------|------------------------------------|
| Calibration window    | data before 2010-01-01 (~5 years)  |
| Test window           | data from 2010-01-01 (~15 years)   |
| MPC horizon $L$       | 7                                  |
| Per-trade cap $\beta$ | 0.5                                |
| Initial capital       | \$1000                             |
| Variance floor $c$    | 1.5                                |
| $\lambda$ grid        | {0.05, 0.1, 0.3, 0.6}              |
| IRW orders            | (1, 2, 3)                          |

Outputs (created on first run):

- `data/m30_main_results.csv`             — per-$\lambda$ trading metrics for CKF and CAST.
- `data/m30_main_results_best_lambda.csv` — best-$\lambda$ summary per (method, dataset).

---

## Running on a Remote Server

The pipeline is **pure CPU** (numpy + scipy LP solver); **no GPU, CUDA, or PyTorch is required**, even when running on a GPU server. A full run over the four datasets takes roughly **40 minutes** on a modern CPU core. Each (dataset, $\lambda$) pair is checkpointed to disk, so an interrupted run is safely resumable.

### Step 1 — Upload the code to the server

From your local machine, copy the entire `CAST/` folder (code **and** data) to the server:

```bash
scp -r CAST/ <user>@<server>:~/
```

Replace `<user>` and `<server>` with your server credentials. Alternatively, clone the repository directly on the server if it is hosted on git / anonymous.4open.science.

### Step 2 — SSH in and enter the project folder

```bash
ssh <user>@<server>
cd ~/CAST
```

### Step 3 — Create a Python virtual environment (Python ≥ 3.10)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

Verify the environment by importing the core modules:

```bash
python3 -c "from src.backtest import metric_row; print('OK')"
```

### Step 4 — Launch the experiment inside `tmux`

Long runs must survive SSH disconnection — use `tmux` (or `screen`/`nohup`):

```bash
tmux new -s cast
python3 -u experiments/m30_main_results.py
```

Detach the session with `Ctrl-b d`; reconnect later with:

```bash
tmux attach -t cast
```

The script prints progress per (dataset, $\lambda$) pair and incrementally writes to `data/m30_main_results.csv` after each pair completes.

### Step 5 — Resume an interrupted run (optional)

If the run is interrupted (network drop, server reboot, etc.), restart it with `--resume` and it will skip every (dataset, $\lambda$) pair already present in the checkpoint CSV:

```bash
python3 -u experiments/m30_main_results.py --resume
```

### Step 6 — Download the results

When the run finishes, copy the result CSVs back to your local machine:

```bash
scp <user>@<server>:~/CAST/data/m30_main_results*.csv ./
```

Two files are produced:

- `m30_main_results.csv`             — 32 rows (4 datasets × 4 $\lambda$ values × {CKF, CAST}).
- `m30_main_results_best_lambda.csv` — 8 rows (best-$\lambda$ row per (method, dataset)).

These match Table I of the paper.

---

## License

Released for review purposes under the terms of the conference submission.
