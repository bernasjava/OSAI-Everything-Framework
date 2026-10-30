<div align="center">

# 🛰️ OSAI Everything Framework

**An AI-driven, multi-agent operating workflow for the OffSec OSAI exam and labs.**

Point it at an authorized host and it enumerates, exploits, escalates, and
documents — keeping a clean, reproducible record of every step.

`Claude Code` · `OffSec OSAI` · `authorized use only` · `MIT`

</div>

> **First and foremost a copilot for the operator** — it drives one box with
> you, suggesting the next move, preparing exact commands, and writing up
> evidence as you go. But when you're busy on another machine, it can also run
> recon, enumeration, and exploitation on its own in the background, doing the
> basic legwork so no target sits idle. You stay in control; it keeps the work
> moving.

> [!WARNING]
> **Authorized use only.** This workflow is for the OffSec OSAI exam and
> labs you are explicitly authorized to test. It is an independent, unofficial
> project and is **not affiliated with or endorsed by OffSec**. You are
> responsible for staying inside your engagement scope and following the
> current exam rules. Never commit exam targets, credentials, flags, or
> reports to a public repository.

---

## What it is

A set of specialized [Claude Code](https://claude.com/claude-code) subagents
coordinated by a single master instruction file. Each agent owns one stage of
the kill chain and writes its findings to shared **session memory**, so the
whole engagement has one durable, reproducible source of truth — even across
dropped sessions.

The technique base is drawn from two field references:

- 📘 [OSCP Everything](https://bernas.gitbook.io/oscp-everything)
- 🤖 [OSAI Everything](https://bernas.gitbook.io/osai-everything)

## Highlights

- **Five focused agents** — recon, web, exploitation, local escalation, and a
  full Active Directory kill chain.
- **One-command autopilot** — run the entire chain across many hosts,
  hands-off, looping on new credentials and pivots.
- **Separation of concerns** — enumeration/detection is kept apart from
  exploitation; destructive actions always pause for the operator.
- **Durable memory** — scope, credentials, the exploitation path, and proofs
  live in `session_memory/` and survive restarts.
- **Report-ready** — `path.md` captures a fully reproducible chain; `proofs.md`
  captures `hostname` + proof file + IP + `whoami` per host and privilege.
- **AI-aware** — covers LLM/agent, RAG, MCP, A2A, and AI-infrastructure
  surfaces alongside classic network, web, and AD targets.

## Workflow

```mermaid
flowchart LR
    O([Operator]) -->|@enumerator IP| E[enumerator]
    E -->|web/API found| W[web_enumerator]
    E --> X[exploitator]
    W --> X
    X -->|shell| L[local_escalator]
    O -.->|manual trigger| AD[ad_reconator_exploitator]
    O ==>|/autopilot| AP{{autopilot}}
    AP -.orchestrates.-> E & W & X & L

    E --> M[(session_memory)]
    W --> M
    X --> M
    L --> M
    AD --> M

    classDef mem fill:#1f2933,stroke:#6b4fbb,color:#fff;
    class M mem;
```

Invoking `@enumerator` alone triggers the automatic chain:
**enumerator → web_enumerator (if web/API) → exploitator → local_escalator.**
Active Directory is always a manual trigger.

## The agents

| Agent | Trigger | Purpose |
|---|---|---|
| 🔍 **enumerator** | `@enumerator <IP>` | Full recon: ports, services, web, API, AI/LLM, AD, cloud. |
| 🌐 **web_enumerator** | auto / `@web_enumerator <URL>` | Deep web & API, with recursive directory/file brute-forcing. |
| 💥 **exploitator** | auto / `@exploitator <IP>` | Finds & exploits vulns → RCE, secrets, or credentials. |
| ⬆️ **local_escalator** | `@local_escalator <IP>` | Local privilege escalation + credential/secret hunting. |
| 🏛️ **ad_reconator_exploitator** | `@ad_reconator_exploitator <DC>` | Full Active Directory enumeration & exploitation. |
| 🚀 **autopilot** | `/autopilot <IPs>` | Runs the whole chain across hosts, unattended. |

## Quick start

```bash
# 1. Clone into your PRIVATE exam workspace (never the public repo)
git clone https://github.com/USER/osai-everything-agent.git
cd osai-everything-agent

# 2. Launch Claude Code from the workspace root
claude

# 3. List the loaded agents
/agents
```

Then add your authorized target IPs to `session_memory/targets.md` — nothing
is attacked unless it is listed there.

### Run a single host

```text
@enumerator 10.10.10.5
```

Enumerates, deep-dives any web service, attempts exploitation, and escalates
if it gets a shell — recording everything to `session_memory/`.

### Run everything, hands-off

```text
/autopilot 10.10.10.5 10.10.10.6 10.10.10.7
/autopilot 10.10.10.5 --ad --until 22:00
```

| Flag | Effect |
|---|---|
| `--ad` | Also run the AD agent against any domain controller found. |
| `--until HH:MM` | Hard stop at the given local time. |

Autopilot stops only for a destructive action, an out-of-scope target, the
deadline, or an exhausted target set — then posts a run summary.

## Repository layout

```text
osai-everything-agent/
├── CLAUDE.md                     # master rules (loaded automatically)
├── README.md
├── USAGE.md                      # operator cheatsheet
├── .claude/
│   ├── agents/                   # the five subagents (@)
│   └── skills/autopilot/         # the /autopilot command
├── session_memory/              # durable state between sessions
│   ├── targets.md                #   scope + per-host state
│   ├── credentials.md            #   every credential / hash
│   ├── path.md                   #   full exploitation chain
│   └── proofs.md                 #   proof per host / privilege
├── scans/                        # raw tool output
└── screenshots/                  # evidence images
```

## Session memory

| File | Holds |
|---|---|
| `targets.md` | Scope and per-host state (`UNTESTED` / `LIKELY` / `CONFIRMED` / `DEAD END`). |
| `credentials.md` | Every credential and hash, its source, and where it works. |
| `path.md` | The complete, reproducible exploitation chain — your report backbone. |
| `proofs.md` | `hostname` + proof file + `ip`/`ipconfig` + `whoami` per host and privilege. |

If a session drops, start a new one from the workspace root and say
"continue" — agents rebuild context from these files.

## Operating principles

- **Scope is sacred** — agents only act on hosts in `targets.md` and never
  pivot out of scope.
- **Evidence is never altered** — raw output is preserved verbatim.
- **Detection ≠ exploitation** — enumeration agents detect and hand off;
  exploitation is deliberate.
- **Operator stays in control** — destructive or irreversible actions pause
  for confirmation.
- **Confidentiality** — exam data never leaves the private workspace.

## Credits & references

Built on the techniques documented in
[OSCP Everything](https://bernas.gitbook.io/oscp-everything) and
[OSAI Everything](https://bernas.gitbook.io/osai-everything).

## License

Released under the [MIT License](LICENSE). This is a personal, unofficial
project shared with the community; it is not affiliated with or endorsed by
OffSec. Candidates must follow the current exam guide and any written
clarification from OffSec.

---

<sub>Inspired by [Zamrax/osai-exam-copilot](https://github.com/Zamrax/osai-exam-copilot).</sub>