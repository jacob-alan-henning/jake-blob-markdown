# AI agents are good now

I wired an agent skill into my blogs code repository and let it run a loop: profile, pick the biggest bottleneck, change one thing, remeasure, keep or revert. In about an hour of running over two days the agent it did real work I would feel good about.

## The loop

Full skill definition: [speed-freak](https://github.com/jacob-alan-henning/skills/blob/main/.claude/skills/speed-freak/SKILL.md).

1. Baseline with benchmarks, CPU/memory profiles, and k6 load tests.
2. Rank bottlenecks by Amdahl-style bound, largest first.
3. Propose a single code change for the current biggest bottleneck.
4. Apply the change, rebuild Docker, run tests/benchmarks/k6.
5. Summarize before/after and decide: keep, revert, or stop.
6. If kept, update the baseline and repeat from step 2.

On each iteration the agent wrote a short markdown report under `/tmp/` with four parts: the current baseline measurements, the proposed code change, the new measurements after running the loop, and a recommended action (`keep`, `revert`, or `stop`). My job was to read that report and choose what to do next.

One change at a time. Never combine changes.

The skill knows:

- Which packages hold the hot paths for metrics and handlers.
- How to run benchmarks and profiles.
- How to run k6 against a local Docker Compose stack that mirrors prod.
- How to run gosec and govulncheck.

It runs on top of the existing CI/CD: tests, static analysis, security scans, Docker builds, and a staging environment that matches the Lightsail nano in production.

## First session

The agent built the Docker image, ran tests, started the container, and waited for health checks to pass.

Baseline benchmarks:

```text
BenchmarkMarkdowntoHtml-10       908 ns/op    112 B/op    2 allocs/op
BenchmarkMetricSnippet-10        362 ns/op     79 B/op    2 allocs/op
BenchmarkPercentileCalc-10      7.53 ns/op      0 B/op    0 allocs/op
BenchmarkMetricsExporter-10      293 ns/op     56 B/op    7 allocs/op
```

Memory profile (by alloc_objects):

```text
17203462 59.52%  MetricsExporter.Export
 3956977 13.69%  writeMetricSnippet
 3866683 13.38%  time.Duration.String
 1933400  6.69%  os.openFileNolog
```

`MetricsExporter.Export` was ~60% of all allocations (17M objects).

k6 at gentle load (20 VUs):

```text
                  p50      p90      p95      p99
Static Files:    3.90     6.97     7.71     9.88
Article List:    3.83     6.20     6.95     10.31
Single Article:  4.02     7.10     8.22     11.10
RSS Feed:        4.21     6.54     7.70     11.82
```

### Iteration 1: remove debug logging

The agent’s first report called out a `telemLogger.Debug().Msgf("bucket: %d count: %d", ...)` inside a histogram loop, called 14 times per export. That meant boxing ints, creating zerolog events, and formatting strings on every export cycle.

One line removed:

```text
MetricsExporter    293 ns/op → 165 ns/op (-44%)
                   56 B/op → 0 B/op (-100%)
                   7 allocs → 0 allocs (-100%)
```

Before:

- `MetricsExporter.Export` dominated heap allocations (about 60% of all objects) and ran at 293 ns/op with 56 B/op and 7 allocs/op.

After:

- `MetricsExporter.Export` dropped to 165 ns/op with 0 B/op and 0 allocs/op and fell out of the heap profile (it had been 262.50MB and 17.2M objects).
- k6 numbers didn’t move, which makes sense: export runs every 5 seconds in the background, not per request.

### Iteration 2: avoid duration string allocation

The next report highlighted `time.Since(s.startTime).String()` allocating a string every call. The agent replaced it with integer arithmetic through `errWriter.int64()` using `strconv.AppendInt` on a stack buffer.

```text
MetricSnippet    79 B/op → 64 B/op (-19%)
                 2 allocs → 1 alloc (-50%)
```

Before:

- `MetricSnippet` allocated 79 B/op with 2 allocs/op; request-path latency was already fine.

After:

- `MetricSnippet` dropped to 64 B/op with 1 alloc/op; k6 latencies stayed effectively unchanged.

We stopped after that.

### First-session summary

Before session 1:

- `MetricsExporter.Export` dominated heap allocations and ran at 293 ns/op with 56 B/op and 7 allocs/op.
- `MetricSnippet` allocated 79 B/op with 2 allocs/op.

After session 1:

- `MetricsExporter`: 293 → 165 ns/op, 56 → 0 B/op, 7 → 0 allocs/op.
- `MetricSnippet`: 79 → 64 B/op, 2 → 1 alloc.
- `PercentileCalc`: ~unchanged, already zero allocs.
- `MarkdowntoHtml`: unchanged.

The exporter went from the main source of heap churn to background noise. GC pressure dropped and CPU smoothed out on the Lightsail nano, buying headroom to add more features and metrics without upgrading.

## Second session

A later run used an updated `speed-freak` skill with an extra step zero: fix the benchmarks so they measure real work.

Changes:

- `BenchmarkMarkdowntoHtml` was measuring a failed `os.Open`; fixed to write and parse a real markdown file.
- `BenchmarkPercentileCalculation` hit a test helper; fixed to benchmark the production `calcPercentiles`.
- `BenchmarkMetricsExporter` had a spin-wait loop; removed so it measures `Export` directly.

With that in place:

- Iteration 1: the agent’s report suggested eliminating the last `writeMetricSnippet` allocation by moving the `[20]byte` buffer around `errWriter`. The follow-up report showed worse escape analysis (more small heap allocs), so it recommended `revert` and the baseline stayed the same.
- Iteration 2: the next report highlighted `mapaccess2_fast64` in `MetricsExporter.Export`. The agent proposed replacing the `boundaryToIndex` map lookup with a `boundaryMsToIndex` switch. Benchmarks in the report went from ~154 ns/op to ~72 ns/op (~53% faster) with 0 allocs/op, and CPU profiles showed `mapaccess2_fast64` disappearing, replaced by an inlined switch.

End-to-end k6 numbers stayed noisy and flat: this is a background exporter that saves tens of nanoseconds every 5 seconds.

### Second-session summary

Before session 2:

- `MetricsExporter` was already down to around 150–165 ns/op with 0 allocations (after session 1), and no longer dominated heap allocations.

After session 2:

- `MetricsExporter` improved again to about 72 ns/op with 0 allocations and no map lookups on the hot path.
- Request-level k6 latencies remained roughly the same; the gains are in background exporter CPU and GC headroom.

### Timeline

- 2026-02-22 — first run: remove exporter debug logging, trim an allocation in the metric snippet, prove the loop works.
- 2026-02-24 — second run: ~40 minutes to fix benchmarks, test and revert a bad idea, and land the map-to-switch exporter change.

## Both sessions together

Before any agent runs:

- `MetricsExporter.Export` ran at 293 ns/op with 56 B/op and 7 allocs/op and accounted for about 60% of all heap allocations.
- `MetricSnippet` allocated 79 B/op with 2 allocs/op.

After both sessions:

- `MetricsExporter.Export` now runs at about 72 ns/op with 0 B/op and 0 allocs/op, with hot-path logging removed and map lookups replaced by an inlined switch.
- `MetricSnippet` sits at 64 B/op with 1 alloc/op.
- Heap churn from the exporter dropped by tens of millions of objects per run, turning it into background noise.

## Are you not impressed?

It’s a perf workflow compressed into a repeatable loop and handed to an agent. I’m getting expert-level tuning in the background, without losing human oversight.

I think it worked well because I had:

A repo-local skill tied to this codebase, Docker compose, benchmarks, and k6 scripts.
A methodology that matches normal perf work: profile, hit the largest bottleneck, change one thing, remeasure, revert if worse.
A backstop of tests, benchmarks, and security checks in the existing CI pipeline.
