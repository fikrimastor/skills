---
name: testcontainers-go-container-leak-diagnosis
description: "Diagnose leaked testcontainers-go containers (e.g. postgres left running after go test): separate the TestMain os.Exit-skips-defer fault from a Ryuk reaper that is disabled or failed, and measure it correctly without mis-timing or mis-filtering."
---

# Diagnosing leaked testcontainers-go containers

Use when containers labeled `org.testcontainers=true` survive a Go test run.

## The fault vs the safety net

`defer c.Terminate(ctx)` followed by `os.Exit(m.Run())` is dead code: `os.Exit`
skips deferred functions. This is a REAL defect in the TestMain (Terminate also
signals the reaper to reap immediately — see `internal/core/docker.go`), but it
is NOT by itself the cause of observable leaks.

Ryuk is the designed safety net for exactly this case. testcontainers-go docs
(`docs/features/garbage_collector.md`): "Even if you do not call Terminate, Ryuk
ensures that the environment will be kept clean." `reaper.go` `Connect()` holds a
TCP conn and `defer conn.Close()` fires on process death, which triggers reaping.
`RyukDisabled` defaults to **false**.

So: report the dead defer as a code-smell/eventual-cleanup fix, NOT as the
demonstrated cause of leaks. Otherwise the user expects leaks to stop and they
don't.

## Measurement traps (each produced a wrong verdict once)

1. **`sleep 2` is too early.** `RYUK_RECONNECTION_TIMEOUT` defaults to 10s
   (`internal/config/config.go`). Any check sooner than that cannot observe the
   difference. Wait >15s after the run exits.
2. **Use ONE consistent filter.** An unfiltered `docker ps -a` counts the reaper
   container and anything else, yielding counts that contradict your own listing
   (e.g. claiming "3 containers" while the label-filtered list shows 1). Use:
   `docker ps -a --filter label=org.testcontainers=true --format '{{.Names}} {{.Image}}'`
   and match the image you actually started.
3. **"No ryuk container" proves nothing.** The reaper container is named
   `reaper_<sessionID>` (`reaper.go:87`) — only the *image* is
   `testcontainers/ryuk` — and Ryuk **self-removes** after reaping. Absence of a
   container matching `ryuk` is expected even when Ryuk ran.
4. **A/B arms share state.** Clean leaked containers between arms, and state the
   expectation relative to the current baseline. A correct-Terminate arm that
   adds ZERO containers is a PASS even if the absolute count doesn't fall — if a
   prior arm leaked one, the baseline is already elevated.

## Decisive experiment (3 arms)

Reproduce the exact TestMain shape in a throwaway module, then:

| Arm | Shape | Ryuk | Expected |
|---|---|---|---|
| A | `defer Terminate` + `os.Exit` | enabled (default) | count returns to baseline after ~10–20s — Ryuk reaped it |
| B | `defer Terminate` + `os.Exit` | `TESTCONTAINERS_RYUK_DISABLED=true` | container persists = genuine leak |
| C | `code := m.Run(); Terminate(); os.Exit(code)` | disabled | adds zero containers = fix works standalone |

Measured result: A = 1→2 at exit→1 after 20s; B = 1→2→2 (leaked); C = baseline→same (no new container).

## The fix (3 lines, per TestMain)

```go
code := m.Run()
_ = pgContainer.Terminate(ctx)
os.Exit(code)
```

Run tests before terminating. Also check whether a leaked container predates the
current run (`docker inspect <name> --format '{{.Created}}'` and its
`org.testcontainers.sessionId` label): a stale survivor with no live `go test`
process is an interrupted run, not evidence of a broken TestMain.
