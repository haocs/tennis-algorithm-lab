# Tennis feedback and evaluation milestones

Design date: 2026-11-02. All implementation items below are planned. The current
deliverable is the [system design](SYSTEM-DESIGN.md) and
[algorithm specification](ALGORITHM-SPEC.md). Training is deferred.

## M0 Freeze contracts

Owners: lab and server, reviewed against website and worker.

- [ ] Fix source/run/clip identities, original-time intervals and revision schemas.
- [ ] Define feedback, annotations, models, predictions, datasets and artifact refs.
- [ ] Agree on activity versus keep/remove labels and target-player policy.
- [ ] Update earlier future-feedback DB wording; retain website likes/courts.
- [ ] Specify additive executor/API changes, TTL, pins and publication compatibility.

Acceptance: synthetic contract examples cover changed boundaries, the same clip in
two runs, time mappings, conflicting revisions and expiration. No private NAS paths
enter source. Existing session manifests and review URLs remain compatible.

## M1 Collect feedback and corrections

Owners: website, server and worker proxy adapter. Depends on M0.

- [ ] Overall tags/comments and start/end early/late flags.
- [ ] On-demand source proxies, context playback, precise corrections and missed-play
      selection outside retained clips, on mobile and desktop.
- [ ] Revision/conflict handling and human/AI provenance, separate from likes.

Acceptance: two visitors cannot silently overwrite each other's edits. Reprocessing
changes clip IDs without losing historical feedback or transferring it silently.
Corrections use original time; proxy expiry preserves labels and offers regeneration.

## M2 Establish benchmark and baseline

Owner: lab, with server dataset snapshots. Depends on M1; overlaps M3 after M0.

- [ ] Review spans from roughly 6–10 varied sessions, including negatives and unclear
      regions; freeze accepted revisions and whole-session development/test splits.
- [ ] Implement duration, action, boundary and fragmentation scoring with explicit
      empty/unknown/zero-denominator behavior.
- [ ] Measure B0 and per-session/condition cost and accuracy; confirm gates before
      tuning candidates.

Acceptance: fixtures reproduce the worked formulas and edge cases. The same release
and scoring version produce reproducible B0 results. Report coverage, failures and
slice support; do not present a small pilot as proven production accuracy.

## M3 Reuse signals and evaluate without rendering

Owners: worker and lab; server extends executor capabilities. Depends on M0.

- [ ] Cache versioned observations/features with validity and time mappings.
- [ ] Implement prediction-only experiments and dependency-based cache keys.
- [ ] Export structured evidence and raw scores separately from editing decisions.
- [ ] Record timing, memory, cache behavior and NAS bytes read/written.

Acceptance: threshold variants reuse pose extraction and produce metrics without
rendering full videos. Extraction changes invalidate only dependent caches.
Existing process and dryrun behavior remains compatible. Restart/cancel/lease-loss
checks prevent duplicate publication or stale results.

## M4 Compare rule candidates

Owners: lab for B1/B2, worker adapters and website comparison UI. Depends on M2/M3.

- [ ] Evaluate tracked-player B1, then temporal-state B2, with bounded tuning and
      feature ablations on development data.
- [ ] Show aligned source-time human/processor rows for selected runs; prioritize
      disagreements and audit random agreements. Render selected contexts only.
- [ ] Evaluate held-out sessions and make an explicit release recommendation.

Acceptance: candidate runs cannot change the benchmark or auto-activate media.
Quality-gate failures remain visible despite a good F2. Comparisons distinguish
old feedback, accepted truth and predictions. Promotion/rollback retains preferences.

## M5 Enforce retention and recovery

Owners: server and worker local cleanup. Depends on M0 registry. Track expiry as soon
as M1/M3 create temporary artifacts; complete enforcement before unattended experiments.

- [ ] Implement TTL/grace queue, registry references, pins and deletion tombstones.
- [ ] Protect active/recovery tasks, serving/rollback sets and dataset/model dependencies.
- [ ] Budget shared caches and implement P2 local scratch cleanup.
- [ ] Show storage/expiry/regeneration status; exercise backup/restore and interrupted
      upload/deletion recovery.

Acceptance: cleanup removes only eligible large artifacts and never labels/history
or originals. Concurrent promotion/pinning cannot lose protected media. Repeated
cleanup after restart is safe. Regeneration produces a linked new attempt. Keep
unattended experiment enqueue disabled until budgets/retention operate.

## M6 Optional AI annotation assistance

Owners: lab specification, worker adapter and website review. Depends on human
benchmark and operational retention.

- [ ] Propose activity spans with versioned model/prompt/context provenance.
- [ ] Measure agreement and human review time; adjudicate disagreements and audit
      agreements. Freeze accepted revisions into new releases.

Acceptance: proposals never overwrite human judgment or silently enter benchmark
truth. Show a review-time benefit at acceptable quality before routine use.
External AI data flow is a separate adapter choice, not required for LAN use.

## Deferred work

- Training classifiers/video networks, GPU scheduling and optimizer checkpoints.
- Ball/racket/contact detection unless a measured failure pattern justifies it.
- Verified Windows deployment, concurrent heavy jobs and automatic preference
  transfer between differently cut clips.

These need separate scope decisions based on measurements. The first useful
delivery is M1 plus M2: actionable corrections and a trustworthy baseline report.
