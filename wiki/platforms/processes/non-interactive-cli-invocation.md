---
id: platforms-processes-non-interactive-cli-invocation
domain: platforms
category: processes
applies_to: [macos, linux]
confidence: verified
sources:
  - https://www.gnu.org/software/coreutils/manual/html_node/nohup-invocation.html
  - https://man.openbsd.org/ssh.1
last_verified: 2026-07-31
related: [platforms-processes-background-services, platforms-shells-portable-shell-scripts, platforms-tools-bsd-vs-gnu-cli, debugging-signals-logs-and-correlation]
---

# Calling an Interactive-Capable CLI from a Script

## When this applies

A script, hook, CI step, or agent session runs another CLI that can also be driven
interactively — an agent CLI in `-p`/`--print` mode, `ssh`, a package manager, a
`git` command that can open an editor or prompt — and the call produces no output
and never returns.

## Do this

1. Wire stdin explicitly at every call site: `cmd </dev/null`. A flag meaning
   "non-interactive" changes how the tool formats output; it does not detach the
   process from the terminal. An inherited TTY stdin stays readable, so any prompt
   the tool decides to show (approval, credential, confirmation) blocks forever
   instead of failing. This is why `nohup` redirects stdin when stdin is a terminal
   "so that terminal sessions do not mistakenly consider the terminal to be used by
   the command", and why `ssh -n` exists and "must be used when ssh is run in the
   background".
2. Pick the wiring by what the tool must read:

| Case | Do |
|------|----|
| The tool needs no input | `cmd </dev/null` |
| The tool must read a payload | Redirect a file: `cmd <payload.txt` — build the file in a prior step so the payload is inspectable after a failure |
| The call must also outlive the session | `nohup cmd </dev/null >"$LOG" 2>&1 & disown` ([platforms-processes-background-services]) |
| The tool is `ssh` inside a script or backgrounded | `ssh -n` (or `-n` plus `-o BatchMode=yes` to fail instead of prompting for credentials) |
| The tool prompts for credentials or approval | Set its non-interactive flag or env var AND close stdin, so a missing credential returns an error |

3. Bound every such call with a hard timeout so a hang fails the step instead of
   stalling the pipeline: `timeout 300 cmd </dev/null` (on macOS the GNU binary is
   `gtimeout` — [platforms-tools-bsd-vs-gnu-cli]).
4. Send both streams to a file for any detached or long call. A hung process with
   zero bytes of output and no log file carries no diagnosis.
5. When the call still hangs with stdin closed and the tool talks to a server, split
   client from server before tuning either side. Read the server's own record of
   the window:

| Server-side evidence | Read it as |
|----------------------|------------|
| No request logged in the window, on a path the server provably always logs | The client never sent it — keep debugging the client side (stdin, auth handshake, DNS, proxy, local queue) |
| The request is logged and still open or long-running | Server-side latency — take it to that service |
| You cannot confirm the path is always logged | Absence is not evidence: send a known-good probe request first and confirm it appears, or corroborate with a request counter ([debugging-signals-logs-and-correlation]) |

## Edge cases

| Case | Then |
|------|------|
| The tool reads stdin as its prompt when stdin is not a TTY | Pass the work as an argument or a file and still close stdin — otherwise it waits on an empty pipe |
| The call hangs only under cron, launchd, or CI | The gap is the minimal environment, not stdin — [platforms-processes-background-services] |
| The tool needs a terminal to run at all (pty-only progress UI) | Allocate a pty deliberately (`script -q`, `ssh -tt`) instead of leaving the inherited terminal attached |
| The call runs as an agent-harness background task | The harness kills its background tasks at session end — detach with `nohup … & disown` |
| The tool exits immediately with an input error after you close stdin | The prompt was real: supply the value through a flag, env var, or credential helper, then keep stdin closed |

## Instead of

| If you are about to | Do this instead | Why |
|---------------------|-----------------|-----|
| Rerun the hanging call with a longer timeout to see whether it is merely slow | Close stdin, add a hard timeout, then check the server's arrival record | A process blocked on a terminal read never finishes, so a longer wait only delays the same unknown |
| Conclude the upstream model, gateway, or network is slow because the client hung | Prove the request left the client (server log entry or request counter) first | A client blocked on stdin sent nothing, so upstream timing explains none of the delay |
| Add the tool's `--yes`/`--non-interactive` flag and call it done | Add the flag AND redirect stdin | The flag suppresses prompts the tool knows about; the inherited TTY still absorbs the ones it does not |

## Sources

- https://www.gnu.org/software/coreutils/manual/html_node/nohup-invocation.html — nohup redirects stdin when it is a terminal, and makes the substitute descriptor unreadable so the command reports an error instead of reading standard input
- https://man.openbsd.org/ssh.1 — `-n`: "Redirects stdin from /dev/null (actually, prevents reading from stdin). This must be used when ssh is run in the background."
