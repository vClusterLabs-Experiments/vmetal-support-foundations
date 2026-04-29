# vmetal Learning Plan — Executable Rewrite Plan

Derived from the multi-LLM pedagogical review (Codex + Claude). Trust the review's findings; this document is the work breakdown, not a re-evaluation.

---

## 1. Phased Work Breakdown

### Phase 0 — Quick Wins (≤30 min each, parallel-safe, no deps)

| ID | File | Change | Acceptance | Effort |
|---|---|---|---|---|
| **P0.1** | `README.md:10,19,21` | Remove "*(optional)*" from M3.5 and M4.5 row entries; keep on M7 only. Update "Optional milestones" line at `:10` to "Only M7 is optional; the `.5` interludes are required." | Grep `optional` shows hits only in M7 row + M7 file | S |
| **P0.2** | `README.md` (after `:5` "How to use this") | Insert standard time-budget block: "Each milestone declares an expected reading and lab budget; ignore links beyond it." | New paragraph present | S |
| **P0.3** | `milestone-01-bmc.md:57` | Delete the apologetic paragraph "Each question has one **Read first** pointer that teaches the underlying concept, plus at most one *Optional* pointer if you want a second angle. The lab itself teaches the rest — don't read every link." (Becomes obsolete after P1 trim.) | Line removed; surrounding text reads cleanly | S |
| **P0.4** | `milestone-01-bmc.md:9` | Strip "asynchronous control contract" bullet from framing list (forward dep into Q4) | Bullet removed; the three remaining framing bullets stand alone | S |
| **P0.5** | All `milestone-*.md` headers | Add line `**Expected: ~Xh reading + ~Yh lab.**` directly under each `**Goal:**` line. Use values from §3 template below. | Every milestone has this line | S |
| **P0.6** | `milestone-01-bmc.md:83-85` | Remove the answer paragraph from Q4 ("It isn't a one-shot RPC. It's a control loop..."). Keep the question; convert answer into a hint at the end: "Hint: think of the caller as a loop, not a one-shot RPC." | Q4 prompt no longer states the answer; the Read-first pointer remains | S |
| **P0.7** | `milestone-03-os-provisioning.md:16` | Same anti-pattern — anchor answer is given. Convert to "Hint:" or remove. | Answer no longer in prompt body | S |
| **P0.8** | `milestone-04-cluster-bootstrap.md:16-24` | Same — anchor answer + dual-stack credentials are spoon-fed. Move to a "Reference" callout below the question rather than as the question's body. | Anchor question reads as a question; reference content moved below | S |

**Phase 0 stop condition:** all 8 tasks land cleanly with no broken cross-refs.

---

### Phase 1 — Milestone 1 rewrite (the anchor problem)

Depends on: Phase 0 (P0.3, P0.4, P0.6 land first to make merging cleaner).

| ID | File | Change | Acceptance | Effort | Dep |
|---|---|---|---|---|---|
| **P1.1** | `milestone-01-bmc.md:3` | Replace overloaded goal sentence with the single-task version: "Drive a Redfish API to power one fake server on/off and read its current state. The contract you exercise here is what every BMC, real or simulated, exposes." | Goal is one sentence, one task | S | — |
| **P1.2** | `milestone-01-bmc.md:7-12` | Trim "What you're actually learning" to 3 bullets: BMC-as-separate-compute-element, Systems resource shape, async control. Drop the "fleet scale" bullet (`:10`). | Bullet count = 3 | S | — |
| **P1.3** | `milestone-01-bmc.md:33-40` | Reading list rewrite: keep OpenBMC host-management + sushy-tools + Lima + libvirt (scaffolding). **Drop** DSP0266 spec link; **replace** DSP2046 schema-guide link with the published mockups index. Annotate each entry with `(read)` or `(reference)`. | List has 4 entries, each tagged | S | — |
| **P1.4** | `milestone-01-bmc.md:55-101` | **Cut Q3** (Systems/Chassis/Managers — defer to M3.5), **Cut Q5** (1,000-node fleet — moves to M4.5), **Cut Q6** (BMC compromise — moves to M7). Keep Q1, Q2, Q4. Renumber 1, 2, 3. | 3 questions remain | M | P0.4, P0.6 |
| **P1.5** | `milestone-01-bmc.md:42-53` | Success criteria trim: keep #1, #2, #3, #5 (power on/off, read state, idempotence). **Move #4** (BootSourceOverride Once vs Continuous) to M2 opening per Codex finding. **Drop #6** as redundant once idempotence is its own line. | 4 success criteria | S | — |
| **P1.6** | `milestone-01-bmc.md:110-118` | Exit artifact: reduce to `notes.md` (3 answers) + `redfish-power.sh` + `scaffolding/`. Drop the `redfish-walkthrough.sh` framing in favor of a simpler power-script. | Artifact list has 3 items | S | P1.4 |
| **P1.7** | `milestone-01-bmc.md` (new) | After Q1 and after Q2, insert a one-line **lab checkpoint** ("Run X. You should see Y. If not, your sushy-tools target isn't bound to the libvirt domain — fix before Q2.") to enforce lab-before-essay ordering. | 2 checkpoints inserted, each with concrete observable | M | P1.4 |
| **P1.8** | `milestone-01-bmc.md:57` | Verify P0.3 deletion is still appropriate post-trim; the apologetic note is now false. | Line gone | S | P0.3 |

**Phase 1 stop condition:** M1 file is ≤90 lines (currently 122), 3 questions, 4 success criteria, ≤5 reading-list entries.

🛑 **STOP-HERE CHECKPOINT #1** — see §5.

---

### Phase 2 — Cross-milestone systemic fixes

| ID | File | Change | Acceptance | Effort | Dep |
|---|---|---|---|---|---|
| **P2.1** | `milestone-02-netboot.md:1-10` | Insert new opening exercise: "Boot override from M1 lives here." Lift M1 SC#4 (`BootSourceOverrideEnabled` Once vs Continuous) into M2 success criteria as a new item #1; renumber existing. | M2 success criteria have 7 items, #1 is boot override | M | P1.5 |
| **P2.2** | `milestone-02-netboot.md:129` | Drop second-link DSP2046 reference; replace with link to the *boot* section of an Ironic Redfish driver doc which is shorter. | DSP2046 referenced only once across plan | S | — |
| **P2.3** | `milestone-03-5-inventory.md` | Add explicit "this milestone is required for M5" sentence in opening line replacing "(optional)". Update goal block. | Opening no longer says optional | S | P0.1 |
| **P2.4** | `milestone-04-5-observability.md` | Same — promote to required. | Opening no longer says optional | S | P0.1 |
| **P2.5** | All milestones with both top-level reading lists AND per-question Read-first pointers (M1, M2, M3, M3.5, M4, M5) | Pick **one** structure per milestone. Recommend: keep per-question pointers, replace top-level reading-list table with a 3-line "Reading budget" callout naming only the canonical doc set. | Each milestone has either a reading-list OR per-question pointers, not both | L | P1.3 |
| **P2.6** | `milestone-04-5-observability.md` | Add the M1-Q5 fleet-scale BMC question here as new Q5 ("1,000-node BMC network — what fails first?"). Cite Brendan Gregg USE method (already in reading list at `:73`). | New Q5 present with Read-first pointer | S | P1.4 |
| **P2.7** | `milestone-07-day2.md` | Add the M1-Q6 BMC compromise question here as a new conceptual Q (likely Q6) under day-2 security context. | New Q present | S | P1.4 |
| **P2.8** | `milestone-03-os-provisioning.md:64-65,73-75,89-92` | Q1/Q2/Q4 reference both Talos and Flatcar. Since `:44` says "pick one and stick with it," reduce dual-stack readings to single-stack with a one-line note: "If you picked Flatcar, swap the Talos pointer for the Flatcar equivalent." | Per-question reading lists each cite one stack as primary | M | — |

---

### Phase 2 addendum (added during execution)

| ID | File | Change | Acceptance | Effort | Dep |
|---|---|---|---|---|---|
| **P2.9** | `milestone-02-netboot.md:23`, `milestone-03-os-provisioning.md:20`, `milestone-04-cluster-bootstrap.md:27`, `milestone-05-capi-metal3.md:20`, `milestone-06-vmetal.md:19` | Rename "Stack changes from MN" / "Stack you're building on top of MN" headers to **"Stack"**. Strip diagram annotations like `<-- unchanged from M2`, `<-- NEW`, `<-- from M5` — they're plan-author shorthand. Rewrite surrounding prose so the diagram reads as a current-state snapshot of *this* milestone, not a diff against the prior one. | Each milestone's stack section is self-contained and reads as "here is your stack now," with no `unchanged` or `NEW` callouts | M | none |
| **P2.10** | `milestone-02-netboot.md:21` | Anchor question's `Answer (write this down once you've proven it)` is the same answer-in-prompt anti-pattern caught in P0.6/0.7/0.8. Convert to hint. | M2 anchor reads as a question | S | (already done during P2 execution) |

**Why these were missed by the original review:** the review pass evaluated content quality but didn't read the milestones in *sequence* the way a learner does. The "Stack changes from MN" framing is invisible at single-file review (each milestone individually has a clear stack diagram); only when a learner sits down at M3 with no fresh memory of M2 does the delta framing become a load problem.

### Phase 3 — Sequencing / structural changes

| ID | File | Change | Acceptance | Effort | Dep |
|---|---|---|---|---|---|
| **P3.1** | `README.md:14-24` | Update milestone table: drop "*(optional)*" from 3.5/4.5 rows. Add column "Time" with `~h` totals for each. | Table reflects new optionality and budgets | S | P0.1, P0.5 |
| **P3.2** | `README.md:26-28` | "Why this order" — add one sentence noting that M3.5 and M4.5 are mandatory load-bearing interludes, not enrichment. | Sentence present | S | P3.1 |
| **P3.3** | `milestone-05-capi-metal3.md:33-34` | Forward-link block currently treats M3.5 as optional. Update to "M3.5 is required reading; this milestone assumes it." | Wording changed | S | P2.3 |
| **P3.4** | `milestone-03-5-inventory.md:25-78` | Inspect Q1-Q4. Are any answerable only after M5 introduces BareMetalHost? If so, mark that question as "deepen after M5" rather than blocking. (Currently Q1 references Metal3 inspection which is M5 territory.) | Forward refs explicitly labeled | M | P2.3 |

🛑 **STOP-HERE CHECKPOINT #2** — see §5.

---

### Phase 4 — Validation pass

| ID | Change | Acceptance | Effort |
|---|---|---|---|
| **P4.1** | Read README.md → M1 → M2 → M3 → M3.5 → M4 → M4.5 → M5 → M6 → M7 in sequence. Note any forward references (M_N references M_>N concepts without flagging). | Forward-reference list compiled; each fixed or annotated | L |
| **P4.2** | Verify all anchor questions are *questions*, not statements with answers. | No anchor question reads like an answer | S |
| **P4.3** | Verify each milestone's exit artifact list is achievable from its own success criteria + reading list (no hidden prereqs). | Artifact ↔ success-criteria mapping verified per milestone | M |
| **P4.4** | Word-count pass: confirm M1 < M2 < M3 in length (foundational milestone shortest). Currently M1=122, M2=155, M3=128 — close. | M1 ≤ 90 lines after Phase 1 | S |
| **P4.5** | Run `grep -n "optional\|skippable" *.md`. Only M7 should match. | Grep result confined to M7 | S |
| **P4.6** | Run `grep -rn "DSP2046\|DSP0266" *.md`. Should appear ≤ once total. | Grep ≤1 hit | S |

🛑 **STOP-HERE CHECKPOINT #3** — see §5.

---

## 2. Milestone-1 sub-plan (standalone checklist)

```
[ ] 1.  Replace L3 goal sentence (P1.1)
[ ] 2.  Trim "What you're actually learning" L7-12 to 3 bullets (P1.2)
[ ] 3.  Drop the "asynchronous control contract" bullet at L9 (P0.4)
[ ] 4.  Reading list L33-40: keep 4 entries, drop DSP0266, replace DSP2046→mockups (P1.3)
[ ] 5.  Success criteria L42-53: keep #1,2,3,5 → renumber to 1-4 (P1.5)
[ ] 6.  MOVE original SC#4 (BootSourceOverride Once/Continuous) into M2 opening (P2.1)
[ ] 7.  Delete L57 "don't read every link" apology paragraph (P0.3)
[ ] 8.  Q3 (Systems/Chassis/Managers L73-79): cut entirely; queue for M3.5 deepen-pass (P1.4)
[ ] 9.  Q4 L83-85: delete the answer paragraph; convert to one-line hint (P0.6)
[ ] 10. Q5 L87-93: cut entirely; move to M4.5 as new Q5 (P1.4 + P2.6)
[ ] 11. Q6 L95-101: cut entirely; move to M7 day-2 (P1.4 + P2.7)
[ ] 12. Renumber surviving Q1, Q2, (former Q4 → new Q3)
[ ] 13. Insert lab checkpoint after each of the 3 questions (P1.7):
        - After Q1: "GET /redfish/v1/Systems/<id> — confirm PowerState reflects virsh state"
        - After Q2: "PATCH boot device, GET it back, confirm value persists across BMC restart"
        - After Q3: "POST Reset, poll PowerState every 1s, observe convergence delay"
[ ] 14. Exit artifact L110-118: notes.md(3 answers) + redfish-power.sh + scaffolding/ (P1.6)
[ ] 15. Re-read top-to-bottom; verify ≤90 lines, no forward refs to Chassis/Managers/scale/security
```

### Proposed new M1 section outline

```markdown
# Milestone 1 — BMC / Redfish

**Goal:** Drive a Redfish API to power one fake server on/off and read its state.
**Expected: ~2h reading + ~3h lab.**

## What you're actually learning
- BMC as a separate compute element (host off ≠ BMC off)
- Redfish Systems resource: how a server appears as an HTTP tree
- Async control: a POST is a request, not a command — state converges later

## Stack (disposable scaffolding)
[unchanged diagram]

## Apple Silicon constraint
[unchanged]

## Reading list (read before lab)
| Topic | Source | Type |
|---|---|---|
| BMC vs host responsibility | OpenBMC host-management | (read) |
| Redfish Systems shape | DMTF mockups, rackmount example | (read) |
| sushy-tools dynamic emulator | OpenStack docs | (read) |
| Lima / libvirt | linked sites | (reference) |

## Success criteria
1. GET /redfish/v1/Systems/ — your fake server appears
2. Read current PowerState
3. Power-on via Redfish action; confirm libvirt domain runs
4. Demonstrate idempotence: any starting state → desired state

## Conceptual questions

### 1. What does a BMC fundamentally provide that the host OS cannot?
**Read first:** OpenBMC host-management
**Lab checkpoint after answering:** GET Systems/<id>, confirm PowerState reflects `virsh list`

### 2. What state does the BMC own vs. the host?
**Read first:** Ironic Redfish driver (boot-mode + virtual-media sections only)
**Lab checkpoint:** PATCH boot device, GET it back, restart sushy-tools, GET again — value persists

### 3. Power actions are asynchronous. How should a consumer handle that?
**Hint:** think loop, not RPC. **Read first:** Kubernetes Controllers concepts
**Lab checkpoint:** POST Reset, poll PowerState every 1s, log the convergence delay

## What is NOT in this milestone
[unchanged]

## Exit artifact
- notes.md (3 conceptual answers)
- redfish-power.sh
- scaffolding/

[Last Updated]
```

---

## 3. Content templates

### 3.1 Standard milestone section order
```
1. Title
2. Goal (one sentence) + Expected time budget line
3. What you're actually learning (3-5 bullets, max)
4. Stack diagram (if changed from prior milestone)
5. Anchor question (a real question, not a question-with-answer)
6. Reading list — single structure: per-question pointers OR top-level table, never both. Each entry tagged (read) or (reference).
7. Success criteria (≤6, lab-verifiable)
8. Conceptual questions, each followed by a Lab checkpoint
9. What is NOT in this milestone
10. Exit artifact
```

### 3.2 Question-design rubric
```
Each conceptual question must:
[ ] be answerable from the listed Read-first material in <30 min
[ ] not state its own answer in the prompt body
[ ] be paired with a lab checkpoint that gives observable evidence
[ ] be at the right Bloom level for milestone position:
    M1-M2: comprehension/application
    M3-M4: application/analysis
    M5+:   analysis/synthesis
[ ] cite scale/fleet questions only after lab supports the inference
```

### 3.3 Reading-list format with budgets
```
| Topic | Source | Type | Time |
|---|---|---|---|
| <one-line topic> | <link> — <which section to read> | (read)/(reference) | ~Nm |
```
Total reading time should match the goal-line budget within ±25%.

### 3.4 Time-budget table (for P0.5)
| Milestone | Reading | Lab |
|---|---|---|
| M1 | 2h | 3h |
| M2 | 4h | 6h |
| M3 | 3h | 5h |
| M3.5 | 2h | 4h |
| M4 | 4h | 6h |
| M4.5 | 2h | 3h |
| M5 | 5h | 8h |
| M6 | 3h | 5h |
| M7 | 3h | 6h (per chosen exercise) |

---

## 4. Risk / tradeoff callouts

| # | Conflict | Recommendation |
|---|---|---|
| **R1** | Codex wants M1 split into M1a + M1b; Claude wants M1 trimmed and boot-override folded into M2. | **Resolve in favor of trim (Claude).** Splitting adds a milestone-numbering ripple (renames, README updates, cross-refs throughout M2-M7); trimming is a single-file change. |
| **R2** | P2.1 lifts M1 SC#4 into M2 *before* P1.5 cuts it from M1 — risk of dangling reference if executed out of order. | Hard sequencing: **P1.5 must complete before P2.1.** Encoded in the dependency column. |
| **R3** | Promoting M3.5 and M4.5 to required (P2.3, P2.4) breaks the README narrative that the curriculum has "skippable interludes." | This is intended; the review found the labeling was misleading. Update README narrative in P3.2 as part of the same change. |
| **R4** | M3.5 Q1 references Metal3 inspection; that's M5 territory. Forward dependency. | P3.4 explicitly addresses with a "deepen after M5" annotation rather than cutting the question. |
| **R5** | Removing the answer-in-prompt at M1 Q4 (P0.6) without M2 immediately reinforcing the control-loop concept means learner may not converge on it. | Q4's lab checkpoint (P1.7) — "POST Reset, poll PowerState every 1s, observe convergence" — *is* the reinforcement. Verify in P4.2. |
| **R6** | M1-Q6 (BMC compromise) moved to M7. M7 is itself optional. Net effect: BMC security is pushed from required → optional. | Acceptable: the original review explicitly said this was a security depth-dive misplaced in a foundational milestone. If concern persists, also put a one-line "BMC is the highest-blast-radius credential in your fleet — return to this in M7" callout in M5 or M4.5. |
| **R7** | P2.5 (deduplicate top-level reading list vs per-question pointers) is L-effort and could create a second wave of cross-reference churn if done carelessly. | Defer P2.5 until after Phase 1 lands and stop-checkpoint #1 confirms M1's structure is the model to copy. |

---

## 5. Stop-here checkpoints

🛑 **STOP #1 — after Phase 1 (M1 rewrite complete)**
Re-read M1 end-to-end and M2 opening end-to-end. Verify:
- M1 < 90 lines, 3 questions, 4 success criteria
- M2's new opening boot-override exercise (P2.1) reads naturally as the *first* thing M2 does
- No mention of Chassis/Managers/fleet/compromise in M1
- A learner with zero context can read M1 + run the lab in 5 hours
**If the through-line feels broken, fix M1↔M2 before proceeding to Phase 2.**

🛑 **STOP #2 — after Phase 3 (sequencing changes complete)**
Re-read README + M3.5 + M4.5 + M5 in order. Verify:
- README's milestone table and "Why this order" prose align
- Promoting M3.5/M4.5 to required has no dangling "(optional)" or "skippable" anywhere
- M5's anchor and reconciliation exercise still hold up given M3.5/M4.5 are now in the path
**If M5 now feels redundant with M3.5/M4.5, that's a real problem — re-evaluate before Phase 4.**

🛑 **STOP #3 — after Phase 4 (validation)**
Read the entire plan in intended order with a stopwatch. Confirm:
- Each milestone fits its declared time budget within ±25%
- Forward references all flagged or fixed
- A new engineer could complete M1-M2 over a weekend and feel they made real progress (not drowning)
**Last gate before declaring the rewrite shippable.**

---

## 6. Minimum-viable-rewrite path (Phase 0 + Phase 1 only)

**What ships:** M1 fully rewritten; cross-cutting hygiene fixes (time budgets, optional-label cleanup, three answer-in-prompt deletions). M2-M7 unchanged in structure.

**Learner experience after MVR:**
- M1 feels right-sized (5h, 3 questions, lab-first). The pain point that motivated the rewrite is fixed.
- M2 still feels heavy compared to the new M1, but its content quality is high — the relative dissonance is a feature, not a bug, because M2 is *where* the boot chain actually lives. Some learners will perceive M2 as "the real start."
- M3.5 and M4.5 are now correctly labeled as required in the README, but their internal phrasing still says "optional" until P2.3/P2.4. **This is the one inconsistency that the MVR ships with.**
- The M1-Q5 (fleet-scale) and Q6 (security) questions are *deleted* but not yet rehomed in M4.5/M7. **Net loss of two questions from the curriculum until Phase 2.** This is acceptable for a 1-2 week MVR window; not acceptable as a long-term state.

**Is MVR shippable?**
- **Yes, with a known-defect note**: cut a one-line addendum to the README ("Phase 2 in flight; M1's scale and security questions temporarily live only in your notes from M4.5/M7 self-study"), and ship.
- **No, if you're publishing this externally:** the M3.5/M4.5 internal-vs-README inconsistency will read as sloppy. In that case, also do P2.3 + P2.4 + P2.6 + P2.7 — about 2 extra hours of work — and call it MVR+.

**Recommendation:** Do MVR+ (Phase 0 + Phase 1 + P2.3, P2.4, P2.6, P2.7). It's the smallest set that leaves the curriculum self-consistent.

---

**Total estimated effort:** Phase 0 (~3h) + Phase 1 (~5h) + Phase 2 (~6h) + Phase 3 (~2h) + Phase 4 (~3h) = **~19h of focused editing** to land the full rewrite. MVR+ is ~10h.