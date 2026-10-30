---
name: autopilot
description: Run the full chain (enumerator → web_enumerator → exploitator → local_escalator, looping on new footholds and credentials) across one or more hosts, going as far as possible without operator interaction. Operator-invoked only.
disable-model-invocation: true
---

# autopilot

One command to run the whole workflow across the hosts the operator gives,
unattended, inside the authorized OffSec OSAI exam environment.

Usage: `/autopilot <host> [host ...] [--ad] [--until <HH:MM>]`
Example: `/autopilot 10.10.10.5 10.10.10.6 10.10.10.7`

Hosts: `$ARGUMENTS`

## Why this is a skill, not a subagent

A subagent cannot start other subagents. The **main session** runs this
skill, and the main session is the only thing that can invoke `enumerator`,
`web_enumerator`, `exploitator`, `local_escalator`, and
`ad_reconator_exploitator`. So when this skill runs, *you* (the main session)
drive the chain directly, launching each subagent in turn and recording its
results in `session_memory/`.

## Operating mode: unattended

Work without stopping for the operator. Follow `CLAUDE.md`'s unattended
rules:

- Take the most reasonable reading and keep going; state each assumption in
  the run log.
- Never pause to ask a question. The only things that stop you are below.
- Record progress continuously in `session_memory/` and the task list so the
  operator can follow along and pick up later.

**Stop and wait for the operator only when:**
- an action would be destructive or irreversible (e.g. the disruptive AD
  CVEs, anything that could take a target down, password changes);
- a target or route falls outside `targets.md` scope;
- `--until <HH:MM>` is set and that local time is reached;
- every host's hypotheses are exhausted (nothing left to try).

Everything the subagents are forbidden to do (out-of-scope hosts, destructive
actions) still applies here.

## Setup (once per run)

1. Confirm each host in `$ARGUMENTS` is in `session_memory/targets.md`. Add
   any that are missing only if the operator listed them as in scope;
   otherwise skip and note it.
2. Create a task-list: one top-level task per host, plus a shared "loot /
   pivot" task.
3. Parse flags: `--ad` enables `ad_reconator_exploitator` against any domain
   controller found (otherwise DCs are only flagged); `--until <HH:MM>` sets
   a hard stop.

## Per-host chain

For each host, in order, run the subagents and fold each result into
`session_memory/` before the next step:

1. **`@enumerator <host>`** — full recon.
   - Record ports/services/web/AI/users in `targets.md`; new creds in
     `credentials.md`.
   - If it found nothing usable, mark the host DEAD END and move on.
2. **`@web_enumerator <url>`** for every web service/API the enumerator
   flagged. Fold results in (paths, endpoints, detection-probe hits,
   hypotheses).
3. **`@exploitator <host>`** — work the ranked hypotheses for RCE / secrets /
   credentials.
   - On a foothold: record the `path.md` step and `proofs.md` entry.
   - On failure: record failed vectors; host stays at its best state.
4. **`@local_escalator <host>`** — only if `exploitator` (or a prior step)
   produced a `CONFIRMED` shell on the host. Escalate and loot.
5. **Domain controller:** if this host is a DC:
   - with `--ad`: run `@ad_reconator_exploitator <host>` (respecting its own
     destructive-action stops);
   - without `--ad`: flag it in the run log and continue.

## Looting and pivoting loop

After each host, feed results back in:

- **New credentials/hashes/keys** → add to `credentials.md`, then re-test
  them against every in-scope host and service (single attempts). A new valid
  login on another host re-queues that host from the appropriate step
  (usually `@exploitator`, or `@enumerator` if it hasn't been enumerated).
- **A pivot route** (tunnel/subnet) that exposes new in-scope hosts → enqueue
  them and run the chain on each.
- Keep looping until no host has an untried hypothesis and no new credential
  opens anything.

Guard against loops: never repeat an unchanged failed step; track which
(host, step) pairs are done; cap retries and note when you hit the cap.

## Ordering and time

- Do quick wins first: hosts with obvious high-value or low-effort vectors
  before slow ones.
- Full UDP scans and long brute-forces run last or in the background so they
  don't block progress.
- Respect `--until`; when the deadline hits, stop cleanly after the current
  subagent returns.

## Reporting

- Keep the task list current (one line per host: current step + best state).
- Record every finding in `session_memory/` as it arrives — don't batch.
- At the end (or at `--until`, or at a stop condition), post a run summary:

```text
AUTOPILOT RUN SUMMARY
Hosts: <list>            Mode: <--ad? --until?>
Per host:
- <IP> — <best outcome: RCE/root/creds/none> — <last step> — <proof refs>
New credentials: <count> (see credentials.md)
Footholds / proofs: <list of proofs.md entries>
Pivots found: <list>
Stopped because: <exhausted | deadline | scope | destructive-action gate>
Needs operator decision:
- <any destructive/out-of-scope/AD-without-flag items waiting>
Suggested next actions:
- <e.g. run with --ad for <DC>, manual review of <finding>>
```

Then stop. Do not take any action that is behind a stop condition without the
operator.