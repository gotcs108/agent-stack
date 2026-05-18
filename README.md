# agent-stack

An open-source, local-first toolkit for building, debugging, and observing AI agents that browse the web. Four repos that compose:

```
┌─────────────────────────────────────────────────────────────┐
│  agent-browser   captures events from a real browser        │
│                  github.com/gotcs108/agent-browser          │
└──────────────────────────┬──────────────────────────────────┘
                           │  writes JSONL trace
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  agent-trace     schema + Python library + CLI              │
│                  github.com/gotcs108/agent-trace            │
└──────────────────────────┬──────────────────────────────────┘
                           │
        ┌──────────────────┴──────────────────┐
        ▼                                     ▼
┌────────────────────┐              ┌────────────────────────┐
│  agent-viewer      │              │  agent-debugger        │
│  browser UI for    │              │  pdb breakpoints on    │
│  one trace         │              │  semantic events       │
│                    │              │                        │
│  gotcs108/         │              │  gotcs108/             │
│  agent-viewer      │              │  agent-debugger        │
└────────────────────┘              └────────────────────────┘
```

All four are MIT-licensed. Pick the parts you need.

## What each repo does

| Repo | What it is | Install |
|------|-----------|---------|
| **[agent-trace](https://github.com/gotcs108/agent-trace)** | The schema. Python `Tracer` class, JSONL on-disk format, CLI replay. | `pip install agent-trace` |
| **[agent-viewer](https://github.com/gotcs108/agent-viewer)** | Single-file HTML page. Drag-drop a trace, see the step timeline. | Open `index.html` |
| **[agent-debugger](https://github.com/gotcs108/agent-debugger)** | Subclass of `Tracer` that breaks (pdb / browser pause) on semantic events. | `pip install agent-debugger` |
| **[agent-browser](https://github.com/gotcs108/agent-browser)** | MV3 Chrome extension that captures real browser sessions in the schema. Plus an RFC for a forked Chromium. | Load unpacked from `extension/` |

## Why this exists

Building agents that run browsers in production fails in a dozen distinct ways: bad credentials, MFA, antibot detection (Cloudflare Turnstile / DataDome / etc.), site structure change, agent decision loops, network timeouts. Without **structured events** that name those modes, every failure-classification system reduces to brittle log-grepping.

This stack imposes a vocabulary at the schema level. `session_end.reason` is an enum: `login_failed_invalid_credentials`, `antibot_blocked_cloudflare_turnstile`, `agent_loop`, and so on. Group a thousand sessions by reason; now you know what's actually broken this week.

The schema is the load-bearing piece. The other three repos exist because you need to *write* events (`agent-browser` captures them; `agent-trace` exposes a Python writer), *read* them (`agent-viewer` for one-off; `agent-trace` CLI for grep), and *intervene* on them (`agent-debugger` breaks the moment a failure mode fires).

## Five-minute tour

```bash
# Write a trace
pip install agent-trace
python -c "
from agent_trace import Tracer
with Tracer(task='demo', model='claude-opus-4-7') as t:
    t.step('Navigate')
    t.nav('about:blank', 'https://example.com')
    t.click('#login', text='Log in')
    t.antibot_detected('cloudflare_turnstile', 0.95, 'iframe present')
    t.end(status='failed', reason='antibot_blocked_cloudflare_turnstile')
print('Trace at:', t.path)
"

# Read it
agent-trace traces/<session-id>.jsonl

# Debug an existing trace
pip install agent-debugger
agent-debugger traces/<session-id>.jsonl --break-on antibot_detected

# Capture from a real browser
# (load extension/ in chrome://extensions, click record, browse, click stop)

# Visualize
# Open agent-viewer/index.html, drag-drop the .jsonl
```

## How this compares to Vercel's Zero

[Vercel Labs Zero](https://github.com/vercel-labs/zero) (released May 15, 2026) is a systems programming language designed so AI agents can read compiler output, parse typed repair metadata, and ship code without parsing human prose. Effects are explicit in function signatures via a `World` capability; `zero check --json` emits stable error codes like `NAM003` with `repair: { id: "declare-missing-symbol" }`; the CLI ships `zero explain` and `zero fix --plan --json` to close the repair loop.

agent-stack is at a **different layer** than Zero. The two are orthogonal — you could build a Zero program that drives a browser and want both.

| Concern | Vercel Zero | agent-stack |
|---------|-------------|-------------|
| **What the agent does with it** | Writes code in it. Reads diagnostics, repairs itself. | Records what it did in a browser. Reads events, classifies failures. |
| **Layer** | Language / compiler | Runtime / browser instrumentation |
| **Surface to the agent** | `zero check --json`, typed diagnostics | `agent_trace.Tracer`, `agent-debugger` breakpoints |
| **Primary failure modes addressed** | Type errors, missing identifiers, effect violations | Antibot detection, login failure, MFA, agent decision loops |
| **Lifecycle phase** | Compile-time + dev loop | Run-time + post-mortem |
| **Format** | JSON diagnostics (stable codes) | JSONL events (stable schema, enum reasons) |
| **License** | Apache 2.0 | MIT |

**Shared design philosophy.** Both stacks apply the same principle — *machines first, humans optional, JSON-structured with stable IDs* — at different layers. Zero applies it to the language/compiler interface; agent-stack applies it to the runtime/observability interface. An agent writing Zero code that drives a browser would use Zero's diagnostics to repair the code and agent-stack's traces to debug what the resulting binary did at runtime.

**The natural bridge.** Zero's effect system declares *what kinds of side effects a function can perform*. agent-stack's schema captures *what side effects actually happened*. A future integration could have Zero programs emit agent-trace events from their `World` capability when they perform browser I/O — making the compile-time effect signature and the run-time event stream literally the same vocabulary. Not built. Tracking the design idea.

Worth noting that Vercel's *broader* agent push (AI SDK + Agents primitive + v0 + AI Gateway + platform observability) is at yet another layer — framework + hosting. agent-stack is also orthogonal to that. None of these three (Zero, AI SDK, agent-stack) is a competitor to either of the others.

## Other comparisons

- **Playwright trace.** Sits below us — captures `Network.*` / `Page.*` events from the browser. We sit above, with semantic event types. Complementary: an agent-trace can reference `playwright-trace.zip` as an artifact.
- **browser-use telemetry.** Free-text logs of the agent's thinking. Useful but unstructured. `agent-trace.Tracer` can be wired into a browser-use loop; v0.2 of agent-trace ships an adapter.
- **LangSmith / Helicone / Langfuse.** LLM-call observability. Captures `llm_call` events well. Doesn't capture browser-side failure modes (antibot vendor, MFA prompt type). Complementary; agent-trace can mirror its `llm_call` events to LangSmith via a sink.
- **Stagehand / Browserbase Director.** Higher-level agent frameworks. Use cases overlap with what writes trace events; agent-stack is the layer that records and analyzes the runs those frameworks produce.

## Status

| Repo | Version | Status |
|------|---------|--------|
| agent-trace | v0.1 | Schema unstable until v1. Python lib + CLI tested. |
| agent-viewer | v0.1 | Single file, working. Renders the v0.1 schema. |
| agent-debugger | v0.1 | Tested. Depends on agent-trace v0.1. |
| agent-browser | v0.1 | Extension working. Forked-browser plan in [docs/RFC.md](https://github.com/gotcs108/agent-browser/blob/main/docs/RFC.md). |

## Roadmap (across the stack)

Near-term (next quarter):
- [ ] **browser-use + Playwright adapters** in agent-trace, so existing agent codebases get traces for one import.
- [ ] **`agent-trace stats`** — aggregate failure-mode distribution across a directory of traces. The 80% use case that no single-trace viewer covers.
- [ ] **Real-time streaming.** Extension POSTs events to `localhost:9999` for live ingest by an agent harness, falling back to download.
- [ ] **OpenTelemetry exporter.** Bridge agent-trace events into OTel for orgs that already have a backend.

Medium-term:
- [ ] **Replay UI for one session.** Side-by-side: trace events + screenshots. Probably forked from rrweb player or Playwright trace viewer.
- [ ] **CDP companion library.** Captures network events the extension API can't reach.
- [ ] **Schema v1.** Stabilize, no more breaking changes.

Long-term:
- [ ] **Forked Chromium.** Per [the RFC](https://github.com/gotcs108/agent-browser/blob/main/docs/RFC.md). Months of work, gated on production validation of the schema first.

## Anti-roadmap

Things we are deliberately not building:
- **A SaaS backend.** The whole point is local-first. JSONL + `grep` + `jq` is the UX.
- **A CRM for agents.** Several adjacent projects are heading there; we're not.
- **Antibot evasion.** This is an observability stack, not a stealth tool. We capture detection signals; we don't fake fingerprints.
- **Auto-redaction by regex.** Pattern matching for emails / credit cards in event data produces false confidence. Caller passes `redact=True` explicitly or it's not redacted.
- **Replacing OpenTelemetry.** Different audience.

## Contributing

Issues and PRs welcome on any repo. The highest-priority contribution at v0.1 is **schema gaps**: if you hit a failure mode that the current event taxonomy can't express, file it on [agent-trace](https://github.com/gotcs108/agent-trace/issues). Each gap shapes the schema toward v1.

## License

MIT across all four repos.
