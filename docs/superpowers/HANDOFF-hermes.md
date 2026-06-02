# Handoff: PieCloak reveal-bug fix + upstream re-sync

**To:** Hermes (fresh agent, zero prior context — read this whole doc before acting)
**From:** prior debugging session, 2026-06-02
**Status:** Root cause confirmed. Relief applied to live server. Fix NOT yet implemented. A full task-by-task plan exists.

---

## 1. Mission (in one paragraph)

PieCloak (`github.com/wsg138/PieCloak`) is an anti-ESP / anti-pie-chart Paper plugin, a **fork** of `github.com/Cubicake/RaycastedAntiESP`. Players (incl. the server owner) see some placed items stuck as **permanent stone blocks**, and **holograms vanish** — intermittently, per-player, unfixable client-side. Your job: **fix the reveal bug by rebuilding the fork's allowlist feature on top of current upstream, restore the full allowlist so those things hide again (but reveal correctly), and re-point the repo to track upstream.** The owner explicitly wants to KEEP hiding holograms/signs — just have them reveal when looked at or stood on. Do NOT "fix" it by permanently removing things from the allowlist.

The full execution plan is at **`docs/superpowers/plans/2026-06-02-piecloak-upstream-resync.md`** — that is your task list. This doc gives you the context to execute it without re-deriving anything.

---

## 2. Repo state (as left for you)

- Local clone: `D:\BadgersMC-Dev\PieCloak`. Working tree **clean**.
- Branch `main` = `origin/main` (the fork), at commit `66cef7a`. **9 commits ahead / 46 behind** `upstream/main`; common ancestor `dbe72d3`.
- Remotes: `origin` → wsg138/PieCloak, `upstream` → Cubicake/RaycastedAntiESP (already added + fetched).
- The plan file and this handoff are under `docs/superpowers/` (committed on branch `handoff/resync`).
- **Live server relief is already applied** by the owner (Track A — trimmed allowlist) and confirmed working. That is a temporary diagnostic mitigation, NOT the fix. When your rebuild ships, Task 6 restores the full allowlist.

The 9 fork-only commits (the feature you must carry forward): `6d1da4d` allowlist filter (additive, 3 new files), `b574aac` gate culling to allowlist (rewrote the 3 hot-path controllers — THIS is the conflict source), `83bc414` diagnostics cmds, `c3b9590` docs, `bb9a4e8` "reveal fixes" (symptom patches — re-evaluate, mostly redundant vs upstream), `2335427` moving-piston filter (likely harmful — see plan Task 7), `f8c54e4` docs, `1b86e64`/`66cef7a` log tweaks.

---

## 3. Confirmed root cause (DO NOT re-litigate — evidence below)

One mechanism, two symptoms. The visibility *decision* is correct; the reveal *delivery* is broken.

- **Decision is fine:** `core/.../raycast/RaycastUtil.java:22` returns visible at point-blank (`total <= alwaysShowRadius`) and for direct LOS (the ray loop stops one step short of the target, so the target's own block never counts as an occluder). So "standing on the hologram" and "looking at the sign" both compute `canSee = true`.
- **Delivery drops the reveal:** when a SHOW transition fires but the tracked state was evicted mid-flight, the reveal is silently skipped:
  - Blocks: `packetevents/.../viewcontrollers/PacketEventsBlockViewController.java`, `processTileEntityTransitions` SHOW case → `if (state == null || state.blockID() == 0) continue;`. The HIDE (→stone) already went out, so the block **stays stone forever**.
  - Entities: `packetevents/.../viewcontrollers/PacketEventsEntityViewController.java:617`, `processEntityTransitions` SHOW case → `if (entity == null …) continue;` logs `show-skipped … missing-entity`. Hologram **stays gone**.
- **Why tracking vanishes = a race the fork is missing the fix for.** Upstream `5ba3140 "Make stale eviction thread safe"` changed eviction from a synchronous call inside the async tick (`evictOldPendingPostSpawnTasks`) to a netty-thread-deferred VarHandle CAS (`markPendingPostSpawnTasksForEviction` → `evictPendingPostSpawnTasksIfRequired`). The fork still does the racy synchronous version. Upstream also reworked mutable tile entities (`d44f3d0`, `3b712d6`, `0166031`, `updateOrInsertTileEntity`) — stops tile `state` going null/zero — and spawn reconciliation (`EntitySpawnTask`, `runPendingPostSpawnTaskForEntity`). The fork branched off `dbe72d3` *right before* all of this and bolted allowlist-gating onto the OLD architecture.

**Conclusion:** rebuilding on `upstream/main` inherits the race fixes → fixes the reveal. Same action also satisfies "get back in the stream."

---

## 4. Why a rebuild, not a rebase

A straight `git rebase upstream/main` conflicts at commit 2/9 (`b574aac`) across all three controllers, because upstream rewrote them. Verified by trial (aborted). The plan instead branches off `upstream/main`, cherry-picks the additive allowlist infra (`6d1da4d`), and **hand-ports** the gating onto upstream's new `handleEntitySpawn0` + block-view code. Port semantics, not lines. Upstream's `handleEntitySpawn` now takes an extra `int entityID` param and delegates to `handleEntitySpawn0` — keep that, thread `shouldCullEntity` through both. Exact code seams are in the plan (Tasks 3–5).

---

## 5. Should this use SPEAR? — No.

SPEAR (spec-proven, EARS requirements, red-green TDD per domain/application layer, hexagonal boundaries, Konsist) is for greenfield/feature work in a codebase we own and can layer cleanly. This task is the opposite:
- It's a **port/merge against a third-party architecture** (PacketEvents controllers), not domain modeling. There are no clean domain/application/infrastructure layers under our control to gate.
- The repo has **zero tests** and the changed code is packet-level, server-dependent logic that can't be exercised by unit TDD. Verification is **compile + in-game integration** (`./gradlew runServer` + `/raesp trace`).
- The session-start check already flagged this is **not a SPEAR project** (no `docs/requirements.md` / `tasks.md`).
- Forcing EARS requirements + `spear:init` here adds ceremony that doesn't map to "re-apply a feature on rewritten upstream code."

The **implementation plan is the right artifact** and already follows the useful disciplines: one genuinely-unit-testable seam gets a real failing test first (Task 2: allowlist key normalization), everything else is gated on compile + the integration protocol in Task 8. Execute the plan as written.

---

## 6. Build / run / verify

- Java 21 toolchain; Gradle multi-module (`locatable-lib`, `logging`, `core`, `packetevents`, `platform-paper`).
- Build the plugin: `./gradlew shadowJar` → `platform-paper/build/libs/RaycastedAntiESP.jar`.
- Isolated test server (DO NOT test on production): `./gradlew runServer` (launches Paper 26.1.2). If the plugin fails to load due to API drift, flip the dev bundle in `platform-paper/build.gradle.kts:19` from `1.21.11-R0.1-SNAPSHOT` to the commented `26.1.2.build.+` line and rebuild.
- **Definition of done (Task 8):** with the FULL allowlist restored — standing on a hologram → visible; direct LOS of a sign → real sign (never stone); out of LOS → hidden (anti-ESP still works); repeat look-away/return ~10× with 2+ players online and see ZERO stranded targets and zero `missing-entity` / `Tile SHOW with missing state` warnings in console.

---

## 7. Gotchas / do-not

- Do NOT permanently trim the allowlist as "the fix" — the owner wants those things hidden, just revealing correctly.
- Do NOT port `bb9a4e8`'s reveal hunks blindly — most are symptom patches for the bug upstream fixed at the source; apply one only if its behavior is still missing after the rebuild.
- `moving_piston` (`2335427`) is likely the cause of stone'd redstone and is not a base-leak vector — default to dropping it (plan Task 7 Step 3).
- Task 5b adds a defensive recovery so a missing-state SHOW re-sends the REAL block instead of leaving stone — keep this even after the race fix.
- Work in a separate worktree (`../PieCloak-resync` off `upstream/main`) per plan Task 0; don't do this on `main`.

---

## 8. First three moves

1. Read the plan: `docs/superpowers/plans/2026-06-02-piecloak-upstream-resync.md`.
2. Execute Task 0 (worktree off `upstream/main`) + Task 1 (cherry-pick `6d1da4d`). Confirm baseline `./gradlew shadowJar` succeeds before porting.
3. Proceed task-by-task. Tasks 3–5 are the real work; re-read each upstream method before editing it.
