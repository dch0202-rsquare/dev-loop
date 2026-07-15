---
id: infrastructure-containers-pid1-log-flushing
domain: infrastructure
category: containers
applies_to: [docker, kubernetes, bash]
confidence: verified
sources:
  - https://mywiki.wooledge.org/ProcessSubstitution
  - https://www.gnu.org/software/bash/manual/bash.html#Process-Substitution
last_verified: 2026-07-15
related: [infrastructure-containers-resource-limits-and-probes, platforms-shells-portable-shell-scripts, infrastructure-observability-logs-metrics-signals]
---

# Container PID1 Losing Buffered Log Output at Exit

## When this applies

A container's main process (PID1) is a bash script that mirrors its own stdout
to a file with process substitution — `exec > >(tee -a "$LOG") 2>&1` — and log
lines go missing from the file: short jobs capture nothing, long jobs lose the
tail (the final summary lines).

## Do this

1. Understand the race: bash does **not** wait for the process inside `>( … )`
   to finish before it exits — the `tee` child is an asynchronous subshell. On a
   normal host the child usually flushes because the shell/terminal lingers, but
   a container is torn down the instant PID1 exits, killing the un-waited `tee`
   mid-flush.
2. Capture the process-substitution PID (`$!`) and drain it in an `EXIT` trap,
   after closing the fds that hold the pipe open:

   ```bash
   exec > >(tee -a "$LOG") 2>&1
   TEE_PID=$!
   trap 'exec 1>&- 2>&-; wait "$TEE_PID"' EXIT
   ```

   Closing fd 1/2 sends EOF to `tee` so it drains and exits; `wait "$TEE_PID"`
   blocks PID1 until it has. (`wait "$!"` on a process substitution works since
   bash 4.4.)
3. Handle SIGTERM so an orchestrator stop still runs the `EXIT` trap — add
   `trap 'exit 143' TERM` (128 + SIGTERM). The default TERM disposition
   terminates bash **without** running the EXIT trap, so a `docker stop` /
   pod eviction skips the flush.

## Edge cases

| Case | Then |
|------|------|
| Short job loses all output, long job loses only the tail | Same cause — the shorter the job, the more of `tee`'s buffer is still in flight at exit; the EXIT-trap `wait` fixes both |
| Script is `sh`/dash, not bash | `$!` for a process substitution is unavailable; append directly with `exec >>"$LOG" 2>&1` (no `tee`), or run the script under bash |
| `tini`/`dumb-init` is PID1 and the script is a child | The child exiting no longer kills the container instantly, but still `wait` the `tee` — a fast container stop can otherwise cut it off |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Trust `exec > >(tee -a log)` to persist output in a container | Save `$!`, close fds, and `wait` it in an `EXIT` trap | The container kills the un-waited `tee` before it flushes |
| Add a longer `sleep` / bigger buffer to "usually" flush | Deterministically `wait` for `tee`'s PID | Teardown timing, not buffer size, is the race |
| Let the default SIGTERM stop the script | `trap 'exit 143' TERM` so the EXIT trap runs | Default TERM skips the flush trap |

## Sources

- https://mywiki.wooledge.org/ProcessSubstitution — "it will continue to run when your script exits (unless you manage your child processes)"; since bash 4.4, `wait "$!"` synchronizes a process substitution. Reproduced on an OrbStack container: no trap → 0/10 runs captured the tail; EXIT-trap `wait` → 10/10
- https://www.gnu.org/software/bash/manual/bash.html#Process-Substitution — process substitution runs asynchronously as a subshell connected via a pipe/FIFO
