<!--
SPDX-FileCopyrightText: 2025 Delos Data Inc
SPDX-License-Identifier: Apache-2.0
-->

# InferenceX sweeps

`launch_sweep.py` runs the profiler OTEL suite once per row of a sweep table, writes each run's
report to its own file, collects every run's numbers into one CSV, and charts them. Use it to
find where a deployment saturates: the concurrency at which throughput stops rising and latency
starts climbing.

`launch_sweep.sh` is the same runner without result extraction or graphing. Both read the same
`sweep.config`, so you can start a sweep with one and resume it with the other.

---

## Before you start

**1. The deployment must already be serving.** The suite drives an existing cluster; it deploys
nothing (`deployment: external: true` in the profile). Bring vLLM up first and confirm it
answers:

```bash
curl -m 5 http://<endpoint.host>:<endpoint.port>/health
```

**2. The profile must describe that deployment.** `profiler_otel/profiles/cai_4n.yaml` declares
the hardware and the coverage the tests assert on. `coverage.hosts` and `coverage.gpus` are
**exact** matches, not floors — a cluster reporting 32 GPUs against a declared 16 fails the same
way as one reporting 8. Check `hardware.gpus_per_machine` against what the launcher actually
uses (`NUM_GPUS_PER_NODE` in the cluster config), not against `nvidia-smi`.

**3. Passwordless SSH to the head node**, for the driver/CUDA versions in each report's
Environment table. Without it the report still generates and says why the versions are missing.

```bash
ssh -o BatchMode=yes username@endpoint.host nvidia-smi --version
```

**4. Environment.** The script defaults these to the hwn-z1 cluster; override any of them:

| Variable | Default | Purpose |
|---|---|---|
| `GPU_INFO_SSH_USER` | `username` | SSH login for the head-node GPU probe |
| `PROMETHEUS_HOST` | `endpoint.host` | where NCCL metrics are read from |
| `GRAFANA_HOST` / `GRAFANA_PORT` | `endpoint.host` / `endpoint.port` | dashboards suite |
| `MODEL` | unset | written to `serving.model` before every row |
| `LABEL` | derived from the model | tag at the front of report filenames |

Do **not** export `LABEL` to a full model id — a `/` in it is a path separator. The script
sanitises it and says so, but the derived default (`openai/gpt-oss-120b` → `gpt_oss_120b`) is
usually what you want, so leave it unset.

---

## The sweep table

`sweep.config` is a plain table. The first non-comment line is the header; each column names a
key the script rewrites in the profile before that row runs.

```
max_concurrency   num_prompts   num_warmups   timeouts.workload
1                 64            2             1800
4                 64            8             1800
...
```

- A **dotted path** (`timeouts.workload`, `serving.model`, `coverage.gpus`) names one exact
  place in the profile. A bare name works too when it is unique in the file.
- Any key in the profile can be a column — `random_input_len`, `tensor_parallel`,
  `timeouts.metrics_available` — so the same machinery sweeps prompt length or parallelism, not
  just concurrency.
- Every row must fill every column; a short row aborts before anything runs, because it would
  otherwise silently inherit the previous row's value.
- An optional `name` column overrides that row's report filename.
- `#` comments and blank lines are ignored.

**The profile is restored on exit** — success, failure, or Ctrl-C — so an interrupted sweep
never leaves a modified `cai_4n.yaml` behind.

---

## Running

```bash
./launch_sweep.py --list            # the table, and the filename each row will produce
./launch_sweep.py --dry-run         # print each row's plan; touch nothing
./launch_sweep.py                   # run every row
uv run --with matplotlib  ./launch_sweep.py --graph           # run, then chart
```

| Flag | Effect |
|---|---|
| `--list` | show the table and exit |
| `--dry-run` | print the plan; no profile edits, no runs |
| `--start N` | resume at row N (1-based, as `--list` numbers them) |
| `--only N` | run just row N |
| `--keep-going` | carry on past a failed row; stop after **2 failures in a row** |
| `--graph` | chart the results |
| `--config PATH` | a different table |
| `--out-dir DIR` | where reports, logs and the CSV go (default `/tmp/cai_sweep_<timestamp>`) |
| `--model ID` | write `serving.model` before every row |
| `--label TAG` | override the derived filename tag |
| `--profile NAME` | a profile other than `cai_4n` |
| `--report-ext {html,md}` | report format |

**Failures.** By default the sweep stops at the first failed row and prints the `--start N` to
resume from. With `--keep-going` it records the row and continues, stopping only after two
consecutive failures — one failed row usually means that row is too heavy for the cluster, but
two in a row means the deployment is down and every remaining row will fail the same way. The
exit code is non-zero whenever any row failed, even if the sweep ran to the end.

**Charting an existing sweep**, without running anything:

```bash
uv run --with matplotlib ./launch_sweep.py --graph --out-dir /tmp/cai_sweep_20260904-140659
```

matplotlib is not a suite dependency, so graphing needs `uv run --with matplotlib` (or install
it once). The script tells you this if the import fails.

---

## What you get

Per row, in the output directory:

| File | Contents |
|---|---|
| `<label>_c<N>_p<N>_w<N>_t<N>.html` | the suite's own report: environment, workload configuration, results, NCCL metric coverage |
| `<label>_c<N>_p<N>_w<N>_t<N>.log` | that row's full pytest console output |

Plus, once per sweep:

| File | Contents |
|---|---|
| `sweep_results.csv` | one row per run: its configuration, `status`, and every measurement |
| `*.png` | the charts, with `--graph` |

`sweep_results.csv` is written even when the sweep aborts — the rows that ran are still hours of
cluster time. Its measurement columns are:

```
requests_completed  benchmark_duration_s  tokens_in  tokens_out
req_per_s  total_tok_per_s  output_tok_per_s
ttft_mean_ms  ttft_p99_ms  tpot_mean_ms  tpot_p99_ms  wall_time_s
```

Two clocks are recorded and they mean different things. `benchmark_duration_s` is
`benchmark_serving.py`'s own timed request phase; `wall_time_s` is the whole containerised run —
image start, model connect, warm-ups, the benchmark, teardown. Throughput is computed against
the former. The gap between them is your per-run overhead.

---

## The charts

Each is a single measure on a single y-axis. Concurrency axes are log2 because the table doubles
per row; a linear axis crushes every early row into the origin.

### `throughput_generated_vs_concurrency.png`
Generated tokens/second against `max_concurrency`. **The primary chart.** The curve rises, bends,
and flattens; the bend is where the backend saturates. If it *falls* after the bend you are past
saturation and into queueing — that is the point to stop, not to push through.

### `throughput_total_vs_concurrency.png`
The same, counting prompt + generated tokens. Separate from the chart above rather than a second
series on it: with fixed input/output lengths, total is exactly `generated × (in+out)/out` — a
constant 9× at 1024/128 — so sharing an axis flattens the generated curve against the baseline
and hides the knee. Read this one when comparing against prefill-heavy figures elsewhere.

### `throughput_generated_vs_num_prompts.png` / `throughput_total_vs_num_prompts.png`
The same two measures against sample size. In a table where `num_prompts` tracks concurrency
these mirror the concurrency charts. They earn their place when concurrency is **held** and only
the sample grows: a flat line means the measurement is stable at that operating point, and a
drift means the run is not in steady state — the sample is too short, or something is degrading
over time.

### `ttft_vs_concurrency.png`
Time to first token, mean and p99, on a log y-axis (it spans milliseconds to minutes across a
full sweep). This is the *cost* of the throughput above. TTFT climbing steeply while throughput
flattens is the definition of saturation: extra concurrency is buying queue depth, not work.

### `tpot_vs_concurrency.png`
Time per output token, mean and p99. Distinguishes two failure modes that look alike in TTFT: if
TPOT stays flat while TTFT explodes, requests are queueing but decode is healthy — the batch is
full and admission is the bottleneck. If TPOT *also* rises, decode itself is degrading, which
points at KV-cache pressure or preemption rather than admission.

### `request_throughput_vs_concurrency.png`
Completed requests/second. Tracks token throughput when output lengths are uniform; diverges
when they are not, and is the number to quote for request-oriented SLOs.

### `scaling_efficiency.png`
Generated tokens/second **per request in flight** — throughput divided by concurrency. Flat means
concurrency is buying throughput linearly. The point where it starts falling is the knee, stated
more sharply than the throughput curve shows it, because a gentle bend there is a clear break
here.

### `throughput_vs_ttft_p99.png`
Throughput against tail latency, log-log, one point per row labelled with its concurrency. **The
operating-point chart.** Pick a tail-latency budget on the x-axis and read off the best
throughput available within it, and which concurrency delivers it. Points crowd vertically at the
left (throughput rising for free) then turn right (latency rising for nothing) — the corner is
where to run.

---


## Troubleshooting

**`FAIL: request failure rate N% exceeds 5% threshold`** — `benchmark_serving.py`'s own gate, not
the suite's. That row offered more load than the deployment could serve and requests timed out
in the queue. It is a legitimate result: that row is past capacity. Use `--keep-going` to record
it and continue.

**`unknown profile 'profiler_otel/profiles/cai_4n'`** — `--workload-profile` takes a *name*
(`cai_4n`), not a path.

**`vLLM server not ready within timeout`** — the readiness probe is pointed at
`endpoint.host`/`endpoint.port` from the profile, which defaults to `localhost`. Set those to the
serving node, or override with `VLLM_HOST`/`VLLM_PORT` passed at the command line.

**`expected N GPU(s) doing work, saw M`** — coverage is an exact match. Either the profile's
`coverage.gpus` is wrong, or the deployment is using a different number of GPUs than declared.

**`GPU driver: not collected -- ssh ...: Permission denied`** — set `GPU_INFO_SSH_USER`, or add
the login to `~/.ssh/config`. Everything else in the report still works.

**Row 12 of a long sweep fails ten hours in** — always start with `--dry-run`, then run rows 1–6
to confirm the shape before committing to the tail. `--start N` resumes; `--keep-going` survives
isolated failures.
