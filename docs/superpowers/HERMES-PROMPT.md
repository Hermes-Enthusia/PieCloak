# Hermes start prompt — PieCloak reveal-fix + upstream re-sync

> Paste everything below the line into Hermes as the opening message. Hermes operates the `Hermes-Enthusia` GitHub account (its own — never BadgersMC's).

---

You are picking up a fully-scoped engineering task. **Read these two docs in the repo first, in order, before doing anything else:**

1. `docs/superpowers/HANDOFF-hermes.md` — context, confirmed root cause (with file:line evidence), build/verify instructions, gotchas. Do NOT re-investigate the root cause; it is settled.
2. `docs/superpowers/plans/2026-06-02-piecloak-upstream-resync.md` — your task-by-task execution plan. Execute it in order.

## Accounts & repos (do not mix these up)

- **p2wn** owns `wsg138/PieCloak` — this is canonical "PieCloak". It is your **PR target** (base). p2wn reviews/merges.
- **BadgersMC** owns `BadgersMC/PieCloak` — a fork of `wsg138/PieCloak` containing these docs. This is your **source**.
- **You (Hermes)** use the `Hermes-Enthusia` GitHub account. **Fork `BadgersMC/PieCloak` into `Hermes-Enthusia/PieCloak`** and do all work there. Authenticate `gh` as `Hermes-Enthusia` only.

## What to do

1. Fork `BadgersMC/PieCloak` → `Hermes-Enthusia/PieCloak`; clone it. Add `upstream` = `https://github.com/Cubicake/RaycastedAntiESP.git` and fetch it (the plan's re-sync rebases onto `upstream/main`).
2. Execute the plan `docs/superpowers/plans/2026-06-02-piecloak-upstream-resync.md` task by task (Tasks 0–9). Key points:
   - It is a **rebuild on `upstream/main`**, not a rebase-in-place. Branch off `upstream/main` in a worktree.
   - Cherry-pick the additive allowlist infra (`6d1da4d`), then hand-port the gating (`b574aac`) onto upstream's rewritten `handleEntitySpawn0` + block controller. Port semantics, not lines — re-read each upstream method before editing.
   - Task 5b adds the defensive SHOW-recovery (no permanent stone on a missing-state reveal). Keep it.
   - Task 6 **restores the FULL allowlist** (holograms/signs/banners/heads hidden again, now revealing correctly). The goal is fix-then-rehide, NOT trimming the allowlist.
   - Verify per Task 8 on the in-repo `./gradlew runServer` test server (Paper). Definition of done: standing on a hologram → visible; direct LOS of a sign → real sign, never stone; out of LOS → hidden; ~10 look-away/return cycles with 2+ players → zero stranded targets, zero `missing-entity` / `Tile SHOW with missing state` warnings.
3. Do NOT use SPEAR for this work — see HANDOFF §5 (it's a port onto third-party packet code with no unit-test surface; verification is compile + in-game).
4. When green, push your branch to `Hermes-Enthusia/PieCloak` and **open a PR with base `wsg138/PieCloak:main`** (p2wn's repo), head `Hermes-Enthusia:<your-branch>`. PR title/body should explain: re-sync onto current upstream + fix the reveal-stranding bug (permanent stone blocks / vanished holograms), and that it inherits upstream's eviction/tile-entity/reconciliation fixes. Reference the plan. Then notify the user that the PR is open for p2wn to review.

## Constraints

- Branch for feature work; never commit straight to `main`.
- Commit per task as the plan specifies; keep commits focused.
- If verification fails or a reveal still strands after the rebuild, STOP and use systematic debugging with `/raesp trace` output — do not patch symptoms or start deleting allowlist entries.
- If the `runServer` plugin load fails on API drift, switch the paperweight dev bundle in `platform-paper/build.gradle.kts:19` to the commented `26.1.2.build.+` line and rebuild (HANDOFF §6).
