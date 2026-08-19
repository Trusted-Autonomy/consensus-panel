# consensus-panel

**Multi-agent consensus algorithms (Raft, Paxos, weighted average) for aggregating N reviewer votes into one score — the piece that sits *before* a policy gate, not instead of one.**

Any pipeline where multiple reviewers (AI agents, human reviewers, or a mix) independently score the same proposal eventually needs to turn N individual scores into one number and one proceed/don't-proceed signal. Do that ad hoc per pipeline and you get subtly different aggregation math in every place it's needed, no consistent way to handle a reviewer that times out, and no answer to "how much do we trust this if the panel actually disagreed" versus "the panel agreed unanimously." This crate is that aggregation step, done once, with the disagreement/timeout handling built in rather than bolted on per call site.

## What it does

Give it a set of reviewer votes, per-role weights, a threshold, and which algorithm to use. It returns an aggregate score, whether the threshold was met, and a per-role breakdown.

```rust
use consensus_panel::{run_consensus, ConsensusAlgorithm, ConsensusInput, ReviewerVote};
use std::collections::HashMap;

let input = ConsensusInput {
    votes: vec![
        ReviewerVote { role: "architect".into(), score: 0.9, findings: vec![], timed_out: false },
        ReviewerVote { role: "security".into(), score: 0.85, findings: vec![], timed_out: false },
    ],
    weights: HashMap::new(),
    threshold: 0.75,
    algorithm: ConsensusAlgorithm::Weighted,
    ..Default::default()
};

let result = run_consensus(&input)?;
// result.score, result.proceed, result.scores_by_role, result.timed_out_roles
# Ok::<(), consensus_panel::ConsensusError>(())
```

## The three algorithms

| Algorithm | When to use it |
|---|---|
| `Weighted` (default fallback) | Simple weighted average, no coordination overhead. Used automatically whenever only one reviewer is active or all reviewers time out — there's nothing to coordinate with one voice. |
| `Raft` | Crash-fault-tolerant, log-persisted, majority-quorum commit. The right choice when the review process itself needs to survive a crash mid-round and resume from a durable log. |
| `Paxos` | Single-decree consensus (prepare/promise/accept/accepted). Suitable when one round of agreement is enough and Raft's multi-round log would be unneeded overhead. |

`run_consensus` auto-degrades to `Weighted` whenever fewer than two reviewers are actually active — coordinating with yourself isn't a coordination problem.

## Persistence is optional

`run_id` and `run_dir` on `ConsensusInput` are `Option<>`. Supply both to get a durable, resumable log (Raft/Paxos) for the run; supply neither and you get pure in-memory computation with no filesystem I/O at all — the right default for a fast scoring pass (e.g. ranking task priority) that doesn't need a crash-recoverable audit trail. `Weighted` never touches either field regardless.

## Audit logging is opt-in

If you want a durable, append-only JSONL record of each consensus run, set `audit_sink: Some(path)` on `ConsensusInput`. Leave it `None` and nothing is written anywhere — this crate makes no assumption about where your application's audit trail should live.

```rust
use consensus_panel::ConsensusInput;
use std::path::PathBuf;

let input = ConsensusInput {
    audit_sink: Some(PathBuf::from(".consensus-panel/audit.jsonl")),
    ..Default::default()
};
```

## What this crate is *not*

**It doesn't decide anything.** `ConsensusResult.proceed` is a threshold check against the aggregate score — it isn't a `Commit`/`Reject`/`Rework`/`Escalate` decision, and consensus-panel has no opinion on what your application should do with a low score. Consensus and policy are a pipeline, not the same interface: N reviewer votes → one aggregated score (this crate) → policy thresholds → a categorical decision (a separate concern).

## Optional: bridging into decision-gate

Enable the `decision-gate` feature to pull in [`decision-gate`](https://github.com/Trusted-Autonomy/decision-gate) and get an adapter that turns a `ConsensusResult` into that crate's `DecisionInput`:

```rust
# #[cfg(feature = "decision-gate")]
# fn example() -> Result<(), consensus_panel::ConsensusError> {
use consensus_panel::{run_consensus, to_decision, ConsensusDecisionPolicy};
use decision_gate::DecisionThresholds;
# use consensus_panel::ConsensusInput;

let result = run_consensus(&ConsensusInput::default())?;
let policy = ConsensusDecisionPolicy::default();
let thresholds = DecisionThresholds::default();

let decision = to_decision(&result, &policy, &thresholds);
// decision_gate::Decision::{Commit, Reject, Rework, Escalate}
# Ok(())
# }
```

The adapter deliberately produces an *honest* `DecisionInput` and nothing more — it doesn't special-case outcomes (no "an active override forces `Commit`", no "a near-threshold score forces `Rework`"). Those precedence rules belong to `decide()`'s own configurable `DecisionThresholds`, not a side-channel in the adapter:

- `verdict`: `Pass` if the panel proceeded with no timeouts, `Warn` otherwise. Never `Block` — a consensus panel has no basis for the kind of hard veto `Block` represents; that's reserved for signals like "this violates an explicit policy," which a scoring panel doesn't evaluate.
- `confidence`: the aggregate score, dampened by how much the panel's individual scores disagreed with each other — a unanimous 0.9 is more trustworthy than a 0.95/0.15 split averaging to the same 0.55.
- `risk_score`: driven by how many reviewers timed out and by that same disagreement, with an active override roughly halving it (an override reflects an informed human decision to proceed despite the panel's raw signal, not evidence the underlying risk disappeared).

The point of routing through `decide()` rather than a bespoke check: every decision your system makes — single-reviewer, rule-matched, or full panel — terminates at the same gate with the same four-outcome vocabulary.

## License

Apache-2.0
