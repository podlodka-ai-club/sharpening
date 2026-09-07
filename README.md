<p align="center">
  <img src="docs/assets/hackathon/hero.svg" alt="swift-claw — recurring tasks that learn from your corrections" width="100%" />
</p>

**swift-claw is a personal AI assistant that runs recurring tasks and turns your corrections into experience for future runs.**

You talk to it in Telegram. It runs on your machine, remembers context, and carries out tasks
on a schedule. The hackathon contribution adds a learning loop: the assistant evaluates results,
derives short lessons from feedback, and tests them on subsequent runs.

<p>
  <a href="#results">Results</a> ·
  <a href="#learning">Learning algorithm</a> ·
  <a href="#architecture">Architecture</a> ·
  <a href="#run">Run it</a> ·
  <a href="https://github.com/podlodka-ai-club/sharpening/issues/1">Hackathon issues</a>
</p>

This repository is the hackathon edition of [swift-claw](https://github.com/ivan-magda/swift-claw).
Its `main` branch contains the self-learning code; the daemon is called **`clawd`**.
The [project dashboard][dashboard] connects milestones, decisions, and reports.

## One task. Several runs. Accumulated experience.

A recurring task should remember your requirements after the conversation ends.
For example, a page monitor should report a meaningful price change
without distracting you every time a counter or delivery estimate updates.

The first benchmark tests exactly this: the agent compares two versions of a page,
identifies material changes, and supplies supporting excerpts.
Here is **an actual pair of responses from the M3 experiment**:

| What changed on the page | Without a lesson | With the derived lesson |
|---|---|---|
| Price: **$12.99 → $9.99** | Report | Report |
| Current delivery estimate: **25 → 18 minutes** | Report | Ignore as a volatile indicator |
| Score under the fixed metric | **85 / 100** | **100 / 100** |

[Original response][example-clean] · [Response with the lesson][example-trained] · [Saved lesson][candidate]

The lesson says to treat frequently refreshed indicators as noise, **unless they reflect
a lasting change in price, availability, obligations, or a deadline**. The rule is stored
for that specific task and becomes part of the context for future runs.

The mechanism applies to recurring tasks in general. Page monitoring is the first measurable
scenario used to test learning; its results and limitations are documented below.

<a id="results"></a>

## What the runs showed

**In M3, the candidate improved the score on three paired checks. It failed the subsequent
check on a new example.** Both are findings from the same experiment.

| Check | Observation | Finding |
|---|---|---|
| Three pairs: clean context → context with a lesson | **85 → 100** in every pair; mean gain of **+15 points** | The lesson corrected false positives on these examples |
| Post-promotion check, active gate | **30 / 100** against a threshold of **90**; `material.missed` | A material change was missed on a new example |
| Check after restart | **Not reached** | This experiment does not establish that the effect survives a restart |

Sources: [three paired comparisons][pairs], [active gate][active], [M3 final JSON report][report].
The run ended with status `incomplete_failed`. Improvement on three pairs does not establish
consistent improvement on new pages.

In this run, an **owner correction** (`owner_correction`) triggered reflection.
The evaluator returned `no_issue` on the initial runs and did not detect the problem:
[algorithm decisions][decisions], [first evaluation][evaluation]. The result therefore demonstrates
learning from owner feedback; autonomous detection of this error has not yet been demonstrated.

The mechanism was integrated into the daemon after the experiment. **All 15 M4 acceptance
criteria were verified at `d784bf91`**: [report with evidence][acceptance]. This confirms the
implementation of the production loop and its contracts. A live comparison between clean and
trained production agents remains a separate demonstration that this report does not replace.

<a id="learning"></a>

## How learning works

<p align="center">
  <img src="docs/assets/hackathon/learning-loop.svg" alt="Loop: run the task, save the result, evaluate and collect feedback, derive lessons, run a trial, then accept or reject" width="100%" />
</p>

1. **Run the task.** Each run receives a pinned version of its lessons and uses it through completion.
2. **Evaluate the result.** Evidence and an evaluation are saved after completion. A technical
   failure does not become a negative quality assessment.
3. **Find a recurring problem.** Two negative results with the same issue code can trigger
   reflection. A single owner correction is enough to start sooner.
4. **Propose lessons.** The model produces a short set of rules for this task.
   Code checks the candidate's size, content, provenance, and whether it is still current.
5. **Test in practice.** The candidate enters a bounded trial. Depending on the results,
   it replaces the stable lessons or is rejected; decisions and their supporting evidence are saved.

**Learning here means updating a task's memory.** Model weights stay unchanged; no fine-tuning is needed.
Lesson versions, evaluations, feedback, and decisions live in SQLite through GRDB.
No external memory service, including xMemory, is used for this.

The evaluator has no tools and cannot see lesson text, the candidate, or trial state.
The reflector receives compatible results, stable lessons, and owner feedback.
This keeps evaluation and proposed changes as separate stages.

## When a lesson becomes stable

<p align="center">
  <img src="docs/assets/hackathon/learning-policy.svg" alt="Bounded candidate trial: up to three runs, two positive results to accept, one negative result to reject, with rollback available" width="100%" />
</p>

| `scheduled-learning/v1` rule | Purpose |
|---|---|
| Last **5 compatible evaluations within 30 days** | Avoid mixing results from different task conditions |
| Up to **3 lessons**, each up to **512 UTF-8 bytes** | Keep experience compact for future runs |
| One trial per task, up to **3 created runs** | Limit exposure to an unproven change |
| **2 positive** results, no negatives or vetoes | Accept the candidate only after all assigned runs resolve |
| **1 negative** result or a hard veto | End the trial while keeping the previous stable lessons |
| **30 days** for assignments, **37 days** for a decision | Close trials even with pauses and missing evaluations |

These deadlines are upper limits: a trial ends earlier when sufficient results are available.

In Telegram, the owner can mark a result **Useful / Not useful**, provide a correction through
**Correct it**, dispute an evaluation, edit or reject a candidate, and roll back a promotion.
Approving a candidate does not count as a positive run result.

Lessons enter context as untrusted data. They cannot change the schedule, recipients,
model, budgets, or tool permissions. The existing policy engine and approval controls
remain in force. Promotion is a heuristic decision; owner confirmations
are recorded separately.

<a id="architecture"></a>

## How it is built

<p align="center">
  <img src="docs/assets/hackathon/architecture.svg" alt="Telegram connects to the Swift daemon: the gateway and scheduler run the agent, while the learning workflow uses shared protocols, the LLM, and GRDB with SQLite" width="100%" />
</p>

**`clawd` is a single Swift 6 daemon with strict concurrency**, running on macOS and Linux.
Telegram is the owner's interface; models are accessed through a configurable provider route.
State stays local, and secrets are encrypted on disk.

| Component | Responsibility | Code |
|---|---|---|
| `ClawCore` | Protocols and types: tasks, evidence, lessons, trials, and decisions | [Learning domain](Sources/ClawCore/Domain/Learning) |
| `ClawAgent` | Task execution, context with pinned lessons, LLM calls, and tools | [Agent runtime](Sources/ClawAgent) |
| `ClawGateway` | Telegram, schedules, feedback, and learning workflow coordination | [Learning services](Sources/ClawGateway/Services/Learning) |
| `ClawData` | Transactions, lesson versions, evidence, promotion, rollback, and retention | [Learning store](Sources/ClawData/Stores/Learning) |

Domain types and protocols are separate from implementations; dependencies are assembled at the
`clawd` root. Pinning lessons to each run, atomic transitions, and state recovery provide
predictable behavior across concurrent events and restarts.

**The experiment controller is written in Python** and is separate from the Swift daemon. It defines
fixed data, run order, and a deterministic scorer. Reference answers
are not passed to the agent with the task.
[Architecture](docs/ARCHITECTURE.md) · [Algorithm specification][algorithm] · [M3 protocol][protocol]

<a id="run"></a>

## Run the hackathon version

You need **Swift 6.3**, macOS 15+ or Linux, a Telegram bot, and LLM access.
On Linux, install `libsqlite3-dev` to build. Build this repository from `main`:

```bash
git clone https://github.com/podlodka-ai-club/sharpening.git
cd sharpening
swift build -c release
export PATH="$PWD/.build/release:$PATH"
```

[Follow the bot setup guide](docs/GETTING_STARTED.md#2-configure): copy the configuration
from `.env.example`, set the Telegram token and LLM route, seal your secrets, and add your
Telegram ID to `CLAW_ALLOWLIST`. The full onboarding guide explains each step.

Enable learning in `~/.swift-claw/clawd.env`:

```bash
CLAW_LEARNING_ENABLED=true
```

Load the configuration into your shell and start the daemon:

```bash
set -a && source ~/.swift-claw/clawd.env && set +a
clawd doctor --check-config
clawd run
```

Learning is off by default; changing the setting requires a restart.
Evaluator and reflector calls use the configured model and shared spending limits,
including the background-task budget.
Use the owner's private chat with the bot for learning.

### Try the loop in Telegram

1. Create a recurring task with `/schedule <task and schedule description>`
   and confirm it. `/schedule list` shows the `jobId`.
2. Run `/runnow <jobId>`, tap **Correct it** below the result,
   and reply with what should change.
3. Open `/learning <jobId>` to inspect lessons and trial state.
4. Run subsequent attempts with `/runnow <jobId>` and rate their results.
   Positive and negative assessments inform the decision under the rules above.
5. After promotion, **Roll back promotion** restores the previous lesson set.
   `/learning reset <jobId>`, with confirmation, starts a new epoch with an empty set.

This is a manual introduction to the production loop. Recorded M3 measurements are available
[separately][report]. Reset stops active use of lessons while preserving history.

## Documentation and evidence

| What to explore | Link |
|---|---|
| Progress and current status | [Parent issue #1](https://github.com/podlodka-ai-club/sharpening/issues/1) · [Dashboard][dashboard] |
| Original experiment and results | [Protocol][protocol] · [Paired checks][pairs] · [Final report][report] |
| Production learning: design and acceptance | [Design][production-design] · [M4: 15 criteria][acceptance] |
| Setup, learning, and owner commands | [Getting started](docs/GETTING_STARTED.md) · [Customization](docs/CUSTOMIZATION.md#scheduled-learning) |
| Other swift-claw capabilities | [Product overview](docs/PRD.md) · [Architecture](docs/ARCHITECTURE.md) |
| Service installation and local development | [Install](docs/INSTALL.md) · [Local development](docs/LOCAL_DEV.md) · [Testing](docs/TESTING.md) |
| Security and contributing | [Security](SECURITY.md) · [Contributing](CONTRIBUTING.md) |

[MIT](LICENSE) © Ivan Magda

[dashboard]: https://github.com/podlodka-ai-club/sharpening/issues/1#issuecomment-5575295475
[acceptance]: https://github.com/podlodka-ai-club/sharpening/issues/6#issuecomment-5575298657
[example-clean]: experiments/scheduled-task-learning/scheduled-learning-v1/results/task-attempts/task-2/result.json
[example-trained]: experiments/scheduled-task-learning/scheduled-learning-v1/results/task-attempts/task-3/result.json
[candidate]: experiments/scheduled-task-learning/scheduled-learning-v1/results/candidate.json
[pairs]: experiments/scheduled-task-learning/scheduled-learning-v1/results/page-adapter-receipt.json
[active]: experiments/scheduled-task-learning/scheduled-learning-v1/results/active-evidence.json
[report]: experiments/scheduled-task-learning/scheduled-learning-v1/results/final-report.json
[decisions]: experiments/scheduled-task-learning/scheduled-learning-v1/results/events/decision-receipts.json
[evaluation]: experiments/scheduled-task-learning/scheduled-learning-v1/results/learning-calls/evaluator-task-0/result.json
[algorithm]: docs/research/170-generic-scheduled-task-learning-algorithm.md
[protocol]: docs/research/172-validation-protocol.md
[production-design]: docs/ARCHITECTURE.md#141-scheduled-learning-terminal-decisions
