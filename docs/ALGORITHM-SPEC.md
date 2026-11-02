# Tennis activity algorithms and evaluation specification

Design date: 2026-11-02. Status: proposed specification, version 1.

The lab owns formulas, labels, signal definitions, candidate specifications and
evaluation procedures. Initial work uses cached poses/features, temporal rules and
bounded tuning. Training is deferred. See [System design](SYSTEM-DESIGN.md) for
storage, APIs and deployment, and [Milestones](MILESTONES.md) for delivery order.

## Labels and editing objective

Activity labels describe stroke attempt, serve, preparation, recovery, ball
collection, waiting/unrelated movement, shadow swing or unclear. Allow coarse
labels when a fine label is uncertain. Missed shots remain legitimate attempts.
Desired-edit labels independently say keep, remove or uncertain.

Initial preference keeps attempts with preparation/recovery, removes downtime and
collection, and configures shadow-swing inclusion. Record the selected target player.
Use temporal spans for the benchmark and complete-action windows for preserving
preparation/swing/follow-through. Stroke peaks and frame/keypoint annotations are
optional later work. Do not treat a single still as proof of ball contact.

## Signals and features

| Signal | Candidate features | Constraints |
| --- | --- | --- |
| Visibility | Torso/wrist confidence, occlusion duration, visible fraction | Missing means unknown, not non-playing |
| Identity | Persistent track, box overlap/displacement, ROI and target selection | Do not independently choose the largest person every frame |
| Arms | Body-relative wrists/elbows, velocity, reach, acceleration | Confidence checks, stable normalization and time gaps |
| Body configuration | Upright/bent posture, knee/hip movement, torso rotation proxies | Camera-view dependent; label these as measurements |
| Movement | Hip/ankle displacement, movement/rest transitions | Pixels are not metres; normalize and account for camera movement |
| Temporal context | Rolling motion, state persistence, pause duration, before/after movement | Use elapsed seconds, not assumed fixed frame counts |
| Recording context | Side/back view, orientation, crop, optional handedness | Unknown categories allowed; inference does not require GPS |
| Optional later evidence | Audio events, ball/racket tracks, court geometry | Add only for a measured gap and benchmark cost/benefit |

Normalize keypoints to the original frame with recorded transforms. Starting
body-relative feature: `r_p(t) = (p(t)-shoulder_center(t))/torso_scale(t)`.
Velocity: `v_p(t) = (r_p(t)-r_p(t_prev))/(t-t_prev)` only when confidence, track
identity, scale and time-gap checks pass. Keep validity masks; never bridge a switch
between people. Window definitions include units, smoothing and missing-data policy.

Use the existing 8-fps extraction profile initially for comparability, not as a
promise of exact boundary precision. Test higher rates on measured failures. Labels
use source PTS, not proxy frame indices. Cache observation extraction separately
from features and from editing so padding changes do not trigger pose inference.

## Candidate and model contracts

| Candidate | Specification | Scope |
| --- | --- | --- |
| B0 | Exact practice-pose-v1 defaults, preprocessing, code/model hashes, including current player selection | Frozen baseline |
| B1 | Existing pose backbone plus persistent target, confidence-aware smoothing and body-relative features | First rule candidate; no training |
| B2 | B1 plus preparation/active/recovery/downtime states, persistence and distinct onset/offset thresholds | Temporal rule candidate after B1 ablations |
| Later classifier | Small classifier or temporal head over cached features | Separate proposal if measured failures justify training |

Every algorithm receives immutable observation/feature references, source duration
and time mapping, target policy and settings. It returns activity intervals/events,
validity/unknown regions, scores with declared semantics and structured evidence.
A separate edit policy applies padding/merge rules to produce ordered,
non-overlapping keep intervals. Unknown handling is explicit, not silently remove.

Model manifest fields: algorithm ID/version, implementation revision, optional
weights/hash, feature/observation schemas, preprocessing, input/output time basis,
settings, backend support, missing-data handling, score semantics and dependencies.
Rule models require no weights. Output retains raw events, final intervals, target
track, visibility and padding/merge reasons. A threshold margin is not a calibrated
probability. A candidate never modifies accepted labels or activates media itself.

## Benchmark construction

Pilot reviewed spans from approximately 6–10 distinct practice sessions, including
side/back views, lighting, missed shots, occlusion, collection, nearby people,
shadow swings, negative-only spans and full transitions. This is a rules/evaluation
pilot, not a claim of sufficient training data. Label some footage without showing
predictions to reduce anchoring.

Freeze source hashes, accepted revisions, desired editing preference and splits.
Exclude unknown/unreviewed time. Keep all adjacent recordings from one practice in
the same split; persist the assignment independently of home-page regrouping and
deduplicate original content. Optionally hold out a court/view separately to assess
generalization beyond familiar conditions.

Tune on development sessions. Freeze scoring/splits before tuning and use held-out
sessions only for final comparison. A test source used for tuning needs replacement
in a new release. Disagreement-heavy samples help debugging but do not replace a
representative benchmark. Report reviewed duration, action count and label coverage.

## Duration formulas

Let D be all reviewed time with accepted keep/remove labels; G is accepted keep
time within D; P is the union of predicted keep intervals intersected with D.
Measure duration |X| in seconds and merge overlapping intervals before counting.

```text
TP = |P intersection G|
FN = |G minus P|
FP = |P minus G|
TN = |D minus (P union G)|

Recall    = TP / (TP + FN)
Precision = TP / (TP + FP)
F2        = 5 TP / (5 TP + 4 FN + FP)
IoU       = TP / (TP + FN + FP)
Downtime seconds per useful minute = 60 FP / TP
```

Recall measures wanted duration preserved; precision measures useful proportion of
retained footage. F2 gives missed seconds four times the denominator penalty of
extra seconds. These are duration metrics, not stroke counts. Accuracy is not the
release criterion because long correctly removed waits can hide missed play.

Zero denominator means null with an explicit reason. Negative-only recordings still
report FP seconds and FP/reviewed-minute. With zero TP, downtime/useful-minute is
null with a zero-useful-output flag. Empty predictions on positive data give recall
and F2 zero, precision null. Do not silently drop negative-only recordings.

## Complete actions and boundaries

For required action window `[s_i,e_i)` and predicted clip `[a_j,b_j)`, action i is
complete if one clip j satisfies `a_j <= s_i + epsilon` and
`b_j >= e_i - epsilon`. Set epsilon to the smaller of one local source frame or
10 percent of the action duration; record actual tolerance. For variable frame rate, use local
frame PTS. Exclude and count actions intersecting unknown annotation time.

```text
Complete-action retention = complete actions / reviewed eligible actions
Missed-or-truncated per 100 actions = 100 (1 - complete-action retention)
```

Duration scoring uses union(P), but action completeness requires a single clip so
fragmenting a stroke cannot receive complete-action credit.

For reference edit `[s,e)` and predicted edit `[a,b)`:

```text
tIoU = max(0, min(b,e) - max(a,s)) / (max(b,e) - min(a,s))
start_error = a - s      positive means too late
end_error   = b - e      negative means too early
```

Match eligible complete edit segments one-to-one by maximum total tIoU with
tIoU >= 0.5. Freeze this rule/version. Do not score boundaries on partially labeled
segments touching unknown time. Report unmatched references/predictions and matched
coverage with boundary statistics; no matches means null, not zero error.

Report signed errors, median/p95 absolute start/end errors and fraction within a
provisional 0.5-second review tolerance. This differs from the one-frame export
allowance. Report extra overlapping fragments per reference and predictions spanning
multiple reference segments as split/merge diagnostics, alongside duration/action
metrics. Boundary precision alone cannot hide missing or merged clips.

## Model comparison and selection

Compare B0 and candidates on the same frozen release and edit preference. Report
pooled duration metrics, equal-weight session summaries, per-session deltas and
side/back/difficult-condition slices. Expose support, unknown coverage and failures.
Once sufficient sessions exist, bootstrap paired deltas by resampling whole practice
sessions; a small pilot remains inconclusive for statistical claims.

Provisional gates: >=98% duration recall and >=98% complete-action retention, with
no material side/back regression. Confirm feasible values, minimum slice support
and allowed regressions after B0 measurement, before candidate tuning. These are
project targets, not industry standards. Insufficient slice data is not a pass.

Among candidates passing quality gates, prefer lower retained downtime and
acceptable boundaries/fragmentation, then compare resource cost. F2 is the summary;
it does not override complete-action failures. Avoid an opaque score mixing
seconds, model bytes and action counts.

```text
Processing ratio = worker end-to-end wall seconds / source duration seconds
Extraction ratio = extraction wall seconds / source duration seconds
Decision ratio   = decision-only wall seconds / source duration seconds
```

Record queue wait separately, plus hardware/backend, cache hit/miss, peak memory,
NAS bytes read/written, artifact size and human review time. Compare cold/warm runs
separately and report rendering/upload apart from analysis. Do not compare an
analysis-only candidate's runtime with a fully rendered baseline's total.

Arithmetic fixtures for 600 seconds of accepted keep time:

| Candidate | TP | FN | FP | Recall | Precision | F2 |
| --- | --- | --- | --- | --- | --- | --- |
| A | 540 | 60 | 30 | 90.0% | 94.74% | 90.91% |
| B | 594 | 6 | 120 | 99.0% | 83.19% | 95.38% |

These validate arithmetic, not performance. Include cases for zero positives,
empty predictions, overlapping predictions, unknown time and fragmented actions.

## Experiment procedure and reports

1. Freeze B0, dataset/revisions/splits, edit policy and metric configuration.
2. Cache extraction and establish baseline intervals equivalent to current B0.
3. Change one signal/rule family at a time; use ablations and bounded development
   tuning. Log failed and no-detection cases rather than filtering them away.
4. Evaluate the selected candidate on held-out sessions without threshold changes.
5. Prioritize disagreements and boundary shifts plus random agreement audits.
   Show aligned source-time rows; render only selected comparison contexts.
6. Record a recommendation; promotion remains explicit and separate from evaluation.

Evaluation manifest pins dataset, annotation, split and scoring versions, candidate
code/model hashes and run IDs. Reports contain overall/session/slice metrics,
action outcomes, boundaries and unmatched counts, split/merge diagnostics, paired
deltas, resource measurements, failures and prioritized review items. Compact
results remain readable after temporary videos expire.

Optional AI annotation follows human feedback and a measured baseline. Store
model/prompt/input provenance, proposals and adjudication. Measure agreement and
review time; audit agreements as well as disagreements. Only accepted revisions
enter new dataset releases. A candidate's own predictions cannot be its independent
test truth. Local execution is the default; an external annotator is a separate
configured data-flow choice, not required for LAN operation.

Training becomes a separate proposal only when a measured failure pattern warrants
it. A small classifier over cached features may be much cheaper than a video
network; deferral is a scope choice, not a claim that all training is expensive.
Scheduling/shared state stay in the server; serving stays in the website.

## References

- [Current detector](https://github.com/haocs/tennis-processing/blob/810690b29df1826f5f144fd8c0882f7938155281/detector.py).
- [Precision recall and F scores](https://scikit-learn.org/stable/modules/generated/sklearn.metrics.precision_recall_fscore_support.html).
- [Temporal localization evaluation](https://activity-net.org/challenges/2018/tasks/anet_localization.html). Matching and gates above are fixed project choices, not a claim to implement ActivityNet mAP.
