# @foothill/logs-benchmark

Score your Log Ingestion and Query Service locally, using the same scorer the
submission platform runs.

## Requirements

- **Node 20+** — no Bun needed
- **Docker** — your service and its database run under Compose

The default runner uses the pinned `grafana/k6:0.54.0` Docker image, so a host
k6 installation is not required.

## Install

```bash
npm install -g ./foothill-logs-benchmark-<version>.tgz
```

## Usage

```bash
# Full local score against your compose stack
logs-benchmark --compose ./docker-compose.yml

# Fast correctness pass only, no load scenario
logs-benchmark --checks-only

# Score a service you are already running yourself
logs-benchmark --endpoint http://localhost:8080
```

Run `logs-benchmark --help` for all flags.

## Resource limits

The CLI applies the same limits the platform enforces — **0.5 CPU / 256 MB** on
your application container and **1 CPU / 1 GB** on Postgres — by generating a
Compose override and layering it over your file. Your own limits are ignored,
exactly as they are during grading.

This is why the CLI orchestrates Compose rather than just pointing at a URL. In
`--endpoint` mode no limits are applied, and the performance number is not
comparable to a graded run.

## How local numbers relate to your grade

**Correctness points match the platform exactly.**

**Performance points are indicative, and they depend on your machine.** The load
generator runs on the same Docker engine and Compose network as the service with
its own CPU budget, which removes the host-versus-VM difference between Linux,
macOS, and Windows. What it cannot remove is core speed: half of a fast core does
considerably more work than half of a slow one, and nothing here normalises for
that. A fast laptop can score twenty points or more above a slower machine or a
loaded CI runner on the *same* submission, so treat the performance, queries, and
reliability numbers as a signal about your code's shape, not as a prediction of
your grade. Quote the reported Docker engine capacity whenever you compare two
runs, and only compare runs from the same machine.

Correctness is the part that transfers. If you want one number to optimise
against, optimise 15/15 checks first: a single failing check caps the whole score
and costs more than the entire queries category.

The benchmark issues aggregate queries through an independent scenario at four
requests per second, so its p95 is measured over enough samples to mean
something. Performance latency and error rate come only from `POST /logs`; query
traffic cannot inflate the ingestion score.

If k6 cannot start every scheduled iteration, the scenario is **retained** and
the CLI warns. That shortfall can mean either that your service applied
backpressure or that the generator ran out of room, and the two are not
distinguishable from the outside — so the number is kept rather than thrown away.
Closing CPU-heavy applications or raising `--generator-cpus` may change it.

Two defaults also differ from the platform, for speed:

|                 | Local default | Platform  | Restore with |
| --------------- | ------------- | --------- | ------------ |
| Scenario length | 0.25x         | 1x        | `--full`     |
| Seeded rows     | 25,000        | 1,000,000 | `--full`     |

`--full` is slow — a 1M-row seed plus full-length stages takes a while.

## Data seed

The CLI uses seed `20260606`. **The graded run uses a different seed.** Service
names, attribute keys, and message text all derive from it, so an index or cache
built around the exact strings you see locally will not help you during grading.

Optimize for the API contract, not for the strings this tool happens to produce.

## Exit codes

| Code | Meaning                                                    |
| ---- | ---------------------------------------------------------- |
| `0`  | The run completed and a score was produced                 |
| `1`  | The run failed — service unhealthy, or a Compose/k6 failure |
| `2`  | Bad usage — unknown flag or invalid value                   |
| `130`| Interrupted — the stack was torn down                      |
