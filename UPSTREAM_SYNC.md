# Upstream sync record (the local/b54-rb* branches)

These branches are not rebased onto every upstream release. Upstream commits are taken one at a time with
`git cherry-pick -x`, so each taken commit's message ends with "(cherry picked from commit <upstream sha>)" and
`git merge-base HEAD upstream/main` keeps naming the last real rebase. This file says which upstream commit the
branch corresponds to. Update it in the same commit as each sync.

Remote: `upstream` = https://github.com/Indras-Mirror/Xolotl.git (`git fetch upstream`).

## Where the branch stands

| | upstream commit | date | |
|---|---|---|---|
| last rebase (the merge base) | `03a138f8` (PR #54) | 2026-10-26 | rb1-rb21 sit on it |
| reviewed up to | `bcaf9c9d` (PR #69) | 2026-10-29 | every commit up to it is taken, deferred or skipped below |

- What upstream has added since the last review:
  `git fetch upstream && git log --reverse --no-merges --format='%h %s' bcaf9c9d..upstream/main`
- The commits taken so far: `git log --format='%h %s' --grep='cherry picked from commit' 03a138f8..HEAD`

## Sync 1 (2026-10-29, rb22): upstream 03a138f8..bcaf9c9d

45 commits, 39 without the merges: 9 taken, 2 taken on rb23 only, 28 skipped.

Taken:

| upstream | local | what | default path |
|---|---|---|---|
| be8f039b | 475a648f | ds41 serve: a partial DSpark batch seed is read from where it was captured | DSpark only |
| 3784d4ff | c81c0a33 | tools/v41/lite: CPU calibration, mixed Q3_K / GPTQ Q2_0 allocation (+ v41_calib.py) | tools only |
| 6d60927a | 04e9cdaf | README / docs: credits, the two-engine CPU and --gpu notes | docs |
| d9ac0c97 | ac49a433 | conversation_copy: the thread-local CUDA streams are destroyed when the thread ends (#57) | a leak fix (layer split) |
| 0f61a239 | bc5a3901 | STRATA_ARENA_THP (opt-in, Linux only) | off |
| 0a224499 | 72aa337d | STRATA_PF_ORDER=arena\|usage (opt-in). Conflict with rb18's HIT_PF ready words: both kept | off |
| 961d390f | a6c8cf1a | STRATA_SERVE_REWIND_GAP (default 16); STRATA_SERVE_BATCH_MIN by arena (16 with a file tier, as before) | aborted turns only |
| b2d0dbb4 | 71d497b7 | ds41-engine.sh container fallback, DS41_PASS_GRAPH_MIB, STRATA_TIER_ORDER=load (opt-in). DS4_TIER_PROF: rb17's own running print kept instead of upstream's (conflict) | off |
| 894d766b | ca898487 | STRATA_SERVE_TURN_SAVE (default on): the turn boundary a request moves past goes to the pool | saves only |

Taken on the branch local/b54-rb23 only (2026-10-29, a candidate until its serve check; it changes the numerics):

| upstream | local | what |
|---|---|---|
| 48b5482c | 56a42759 | the vendored ggml moves to llama.cpp 6050b467 (825 commits): upstream's new xolotl-ggml.patch, as is |
| a7523418 | 112c0ee2 | fetch-llama.sh pins 6050b467 |

The commit after them (local rb23) re-applies this tree's own ggml changes (rb8g, rb10-rb14 and rb16: the ops
DSV41_KV_GATHER / ROUTE / HOOK / STREAM, the FP4 packers, the stream hooks) on that tree with a three-way merge, and
writes the patch again. Conflicts: argsort.cuh (both declarations kept), ggml.c (GGML_OP_COUNT 104 + 4 = 108), and
top-k.cu, where llama.cpp now batches many-row top-k itself (radix / bitonic): its version replaces rb8e's
(GGML_CUDA_TOPK_PER_ROW is gone).

Skipped (the MiniMax M3.1 port: tools/m31, docs/MINIMAX*, its CMake option; nothing DeepSeek):
41c46c2f b7121972 45b54299 e28d89b8 060046f3 0bbf7f09 3c631777 dbb15cc3 616a33db b810557f 42f5a1c4 5cdfc335
a67f1b7e f44fe3c3 b510efdd bd819920 693c2f8e 14de1a89 ac679f08 f5b2324c e5cf56e9 b48f8cca 9199afe1 25cf02f0
cb8b3a56 ff133673 70af23b5 481daf73

Checked (2026-10-29 evening, the serve scenarios on K:\model\test's DeepSeek config: 11 text scenarios, the cancel pair
and two pictures; engine pinned/ds41_generate-rb22.exe, sha256 31d3f0fc..., built with CUDA 13.4):

- rb22 gives rb21's answers in all 13 (a warm rb21 run next to it; decode 65.42 vs 65.17 tok/s), with TURN_SAVE and
  REWIND_GAP on or off, and the same answers in all four rb22 runs.  An rb21 run started cold after an idle hour
  (decode 59, the 15.7K prompt 14.4 s instead of 10.1) differed from code1 on: the slow state's timing, not rb22.
- STRATA_PF_ORDER=arena: -1.5 % decode in an ABBA (66.80 / 65.79 / 67.25 / 68.22 tok/s), other placements and so other
  answers.  Not used.
- rb23 (llama.cpp 6050b467; pinned/ds41_generate-rb23b.exe, sha256 601decfb..., after the top-k fix below): the same
  fidelity to the original as rb22 on the Qwen KLD texts (16 chunks, the prompt path: EAR 86.09 / 86.07 %, KLD 0.1620 /
  0.1626, PPL ratio 1.0526 / 1.0529; 4 chunks, decode with the live route bias: EAR 84.69 / 84.66 %); rb22 -> rb23 is KLD
  0.0098, a sixteenth of the quantization's 0.16.  Decode 0 to +2 % (two ABBAs: +0.2 %, +2.4 % with a cold first rb22
  run), prompts the same (15.7K 10.4 s).  Other answers than rb22 from a few tokens on (the numerics).
- The first rb23 build gave other answers run to run from the 15.7K prompt on: llama.cpp's batched top-k gathers the
  ties of the k-th value in atomic order.  rb8e's in-order tie gather is back on its radix path (the commit before this
  note); since then two rb23 runs give the same 13 answers.
- TURN_SAVE (rb22, the history-edit check): a 9.3K-token system prompt, three turns, then turn 2's question rewritten:
  9,299 tokens reused, 145 read in 1.83 s, against everything re-read (9,444 tokens, 7.74 s) with it off; the same answer.
