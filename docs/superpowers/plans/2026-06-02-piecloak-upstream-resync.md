# PieCloak Upstream Re-Sync + Reveal-Bug Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the reveal-stranding bug (allowlisted things stay stone / stay hidden even with direct line-of-sight or while standing on them) by re-basing PieCloak's allowlist feature onto current `upstream/main`, inheriting upstream's eviction/tile-entity/reconciliation fixes. Then **restore the full allowlist** (holograms, signs, banners, etc. are hidden again — but now reveal correctly). Then re-point the repo to track upstream cleanly.

**Confirmed root cause (do not re-litigate):** The raycast/visibility decision is correct — `RaycastUtil.raycast` returns `true` (visible) at point-blank range (`total <= alwaysShowRadius`) and for direct LOS. The bug is in *delivering* the reveal. When a SHOW transition fires but the tracked state was evicted mid-flight, the reveal is silently skipped: `processTileEntityTransitions` does `if (state == null || state.blockID() == 0) continue;` (block stays stone) and `processEntityTransitions` does `if (entity == null …) continue;` (hologram stays gone). The HIDE was already sent, so the target is stranded permanently. The eviction race is fixed upstream by `5ba3140` (defer eviction to the netty thread via VarHandle CAS instead of racing the async tick) plus the mutable-tile-entity rework (`d44f3d0`/`3b712d6`/`0166031`/`updateOrInsertTileEntity`) and spawn-reconciliation commits.

**Architecture:** Do NOT rebase in place (a straight `git rebase upstream/main` conflicts from commit 2/9 because upstream rewrote all three hot-path controllers). Instead, **rebuild on upstream**: start a branch off `upstream/main`, cherry-pick the *additive* allowlist infrastructure, then hand-port the gating logic onto upstream's new `handleEntitySpawn0` / block-view architecture. The gating philosophy is unchanged — "managed" = allowlisted = inserted into an anti-ESP view; everything else passes through untouched. A defensive SHOW-recovery (Task 5b) guards against any residual race.

**Tech Stack:** Java 21 toolchain (Gradle multi-module: `locatable-lib`, `logging`, `core`, `packetevents`, `platform-paper`), Paper 1.21.11 dev bundle (compile) / Paper 26.1.2 (`runServer`), PacketEvents 2.12.0, Configurate 4.2.0, Gradle Shadow. Build artifact: `platform-paper/build/libs/RaycastedAntiESP.jar` via `./gradlew shadowJar`.

---

## Reality check on testing (read before starting)

This repo has **zero automated tests** (`git ls-files | grep src/test` → 0) and the changed code is packet-level Paper plugin logic that depends on a live server, Bukkit's Material registry, and PacketEvents. Pure unit TDD does not fit the hot-path controllers. Therefore:

- **Primary verification = compile + integration.** Every code task ends with `./gradlew shadowJar` (must compile) and, at phase boundaries, an in-game check via the built-in `./gradlew runServer` task (isolated Paper 26.1.2 server — never test against production).
- **One genuinely unit-testable seam exists:** target-name normalization (`minecraft:` prefix stripping + group parsing) in `TargetFilterConfig` / the filter. Task 2 adds the project's first JUnit test for it. If wiring JUnit into the `core` module proves heavier than ~15 min, downgrade it to an integration check and note that in the task — do not let test-infra setup balloon the plan.

**Known gotcha:** `platform-paper/build.gradle.kts` compiles against `paperDevBundle("1.21.11-R0.1-SNAPSHOT")` (line 19) but `runServer` launches `minecraftVersion("26.1.2")` (line 91). If `runServer` fails to load the plugin due to API drift, switch the dev bundle to the commented `26.1.2.build.+` line and recompile. Decide per-run; do not change it preemptively.

---

## File Structure

**Cherry-picked unchanged from fork commit `6d1da4d` (additive — new files, expected to apply clean):**
- `core/.../core/config/TargetFilterConfig.java` — allowlist config record + `Mode` enum.
- `packetevents/.../packetevents/target/PacketEventsTargetFilter.java` — `shouldCullEntity(EntityType, isPlayer)` decision + `DISABLED` instance.
- `platform-paper/.../paper/target/PaperTargetFilterService.java` — expands material groups via Bukkit registry, resolves entity/tile-entity allowlists.

**Hand-ported onto upstream's new architecture (these WILL conflict — port by hand):**
- `core/.../core/view/controller/PacketEntityViewController.java` — add `shouldCullEntity` gate to `handleEntitySpawn0`; add `isManagedEntity`/`hasManagedEntity`; relax passenger/leash handlers for unmanaged related entities.
- `packetevents/.../viewcontrollers/PacketEventsEntityViewController.java` — compute `shouldCullEntity` at the `SPAWN_ENTITY` seam, pass through new signature.
- `packetevents/.../viewcontrollers/PacketEventsBlockViewController.java` — gate tile-entity insertion to allowlisted block IDs.
- `platform-paper/.../paper/packets/PaperPacketEventsEntityViewController.java` + `PaperPacketEventsBlockViewController.java` — wire the `PaperTargetFilterService` into the controllers' constructors.
- `platform-paper/.../paper/RaycastedAntiESP.java` — construct + inject the target filter service on enable.

**Config (already trimmed in Track A — re-apply on the new branch):**
- `platform-paper/src/main/resources/config.yml` — `target-filter` block with decorative targets removed.

**Optional fork extras (evaluate, port only if still relevant):** diagnostics commands (`83bc414`), visibility tracing + reveal patch (`bb9a4e8` — much of this may be redundant against upstream's reveal fixes; port the *trace* command but re-evaluate the *fix* hunks), moving-piston filter (`2335427`), unknown-sync/passenger logging (`66cef7a`, `1b86e64`).

---

## Task 0: Create the re-sync worktree

**Files:** none (git setup)

- [ ] **Step 1: Confirm remotes + fetch**

Run:
```bash
cd /d/BadgersMC-Dev/PieCloak
git remote -v            # expect origin=wsg138/PieCloak, upstream=Cubicake/RaycastedAntiESP
git fetch upstream --prune
```
Expected: `upstream/main` present and current.

- [ ] **Step 2: Create an isolated worktree off upstream/main**

Run:
```bash
git worktree add -b resync/upstream-allowlist ../PieCloak-resync upstream/main
cd ../PieCloak-resync
git log --oneline -1     # expect the upstream/main HEAD (e53c510 "Merge pull request #55 ...")
```
Expected: HEAD is upstream tip, working tree clean. All later tasks run in `../PieCloak-resync`.

- [ ] **Step 3: Baseline build (prove upstream compiles before we touch it)**

Run: `./gradlew shadowJar`
Expected: BUILD SUCCESSFUL; `platform-paper/build/libs/RaycastedAntiESP.jar` exists. If this fails, STOP — fix the environment before porting (see "Known gotcha").

- [ ] **Step 4: Commit nothing yet** — Task 0 is environment only.

---

## Task 1: Land the additive allowlist infrastructure

**Files:**
- Create (via cherry-pick): the 3 files listed under "Cherry-picked unchanged" above.

- [ ] **Step 1: Cherry-pick the allowlist infra commit**

Run:
```bash
git cherry-pick -x 6d1da4d
```
Expected: clean apply (all three are new files). If it conflicts because a file path now exists upstream, open the conflict, keep the fork version, `git add` it, `git cherry-pick --continue`.

- [ ] **Step 2: Compile**

Run: `./gradlew :core:compileJava :packetevents:compileJava`
Expected: `core` compiles. `packetevents` / `platform-paper` may FAIL here because `PacketEventsTargetFilter` is referenced by controllers we haven't ported yet — that is fine at this step; only `core` must compile clean. Note which references are missing; they become the port checklist for Tasks 3–5.

- [ ] **Step 3: Commit (cherry-pick already created it)** — verify with `git log --oneline -1` shows the allowlist commit. No extra commit needed.

---

## Task 2: Unit-test the allowlist matcher (project's first test)

**Files:**
- Modify: `core/build.gradle.kts` (add JUnit 5 test deps + `test { useJUnitPlatform() }`)
- Test: `core/src/test/java/games/cubi/raycastedantiesp/core/config/TargetFilterConfigTest.java`

- [ ] **Step 1: Add JUnit to the core module**

Add to `core/build.gradle.kts` dependencies:
```kotlin
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
```
And after the `dependencies { }` block:
```kotlin
tasks.test { useJUnitPlatform() }
```

- [ ] **Step 2: Write the failing test**

`core/src/test/java/games/cubi/raycastedantiesp/core/config/TargetFilterConfigTest.java`:
```java
package games.cubi.raycastedantiesp.core.config;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;

class TargetFilterConfigTest {
    @Test
    void normalizesBareAndNamespacedKeysIdentically() {
        // Both forms must resolve to the same canonical key so config authors
        // can write "villager" or "minecraft:villager" interchangeably.
        assertEquals(
            TargetFilterConfig.normalizeKey("villager"),
            TargetFilterConfig.normalizeKey("minecraft:villager"));
    }
}
```
> If `TargetFilterConfig` performs normalization inline rather than via a static `normalizeKey`, extract that logic into a `static String normalizeKey(String raw)` method first (pure refactor, no behavior change), then point the test at it.

- [ ] **Step 3: Run the test, verify it fails**

Run: `./gradlew :core:test --tests TargetFilterConfigTest`
Expected: FAIL — `normalizeKey` not defined (or assertion fails if logic is wrong).

- [ ] **Step 4: Implement `normalizeKey` in `TargetFilterConfig`**

```java
public static String normalizeKey(String raw) {
    String s = raw.trim().toLowerCase();
    return s.startsWith("minecraft:") ? s.substring("minecraft:".length()) : s;
}
```

- [ ] **Step 5: Run the test, verify it passes**

Run: `./gradlew :core:test --tests TargetFilterConfigTest`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add core/build.gradle.kts core/src/test/
git commit -m "test: add JUnit + allowlist key-normalization test"
```

---

## Task 3: Port the entity-spawn gate onto upstream's handleEntitySpawn0

**Files:**
- Modify: `core/.../core/view/controller/PacketEntityViewController.java`

Upstream's current shape (do not delete the new `entityID` param or the `runPendingPostSpawnTaskForEntity` call):
```java
protected boolean handleEntitySpawn(P packet, int entityID, boolean isPlayer, PlayerData playerData, UUID world, int currentTick) {
    boolean returnValue = handleEntitySpawn0(packet, isPlayer, playerData, world, currentTick);
    playerData.nettyData().runPendingPostSpawnTaskForEntity(entityID);
    return returnValue;
}
```

- [ ] **Step 1: Thread `shouldCullEntity` through both signatures**

Change to:
```java
protected boolean handleEntitySpawn(P packet, int entityID, boolean isPlayer, boolean shouldCullEntity, PlayerData playerData, UUID world, int currentTick) {
    boolean returnValue = handleEntitySpawn0(packet, isPlayer, shouldCullEntity, playerData, world, currentTick);
    playerData.nettyData().runPendingPostSpawnTaskForEntity(entityID);
    return returnValue;
}
```

- [ ] **Step 2: Re-apply the gate inside `handleEntitySpawn0`** (port of fork commit `b574aac`)

Replace the body of `handleEntitySpawn0` with:
```java
protected boolean handleEntitySpawn0(P packet, boolean isPlayer, boolean shouldCullEntity, PlayerData playerData, UUID world, int currentTick) {
    if (world == null) {
        Logger.error(new RuntimeException("World null when handling spawn entity packet, uuid=" + playerData.getPlayerUUID() + " tick=" + currentTick), 2, PacketEntityViewController.class);
        return false;
    }

    NettyEntityLocatable<?,?> entity = processEntitySpawn(playerData, packet, world, currentTick);
    if (entity == null) {
        return false;
    }

    // Players are always tracked in the player view (gating never hides players here).
    if (isPlayer) {
        entity.setVisible(true);
        entity.setClientVisible(true);
        insertEntityToPlayerView(entity, playerData);
        return false;
    }

    // Non-allowlisted entities pass through untouched and are NOT inserted into any anti-ESP view.
    if (!shouldCullEntity) {
        entity.setVisible(true);
        entity.setClientVisible(true);
        return false;
    }

    if (entityConfig.enabled()) {
        double distanceSquared = playerData.ownLocation().distanceSquared(entity);
        if (distanceSquared > hideOnSpawnEntityDistanceSquared) {
            entity.setVisible(false);
            entity.setClientVisible(false);
            insertEntityToEntityView(entity, playerData);
            return true;
        }
    } else {
        entity.setClientVisible(true);
    }
    insertEntityToEntityView(entity, playerData);
    return false;
}
```

- [ ] **Step 3: Add the managed-entity helpers if absent** (port from `b574aac`)

If `isManagedEntity` does not already exist upstream, add:
```java
protected boolean isManagedEntity(int entityID, PlayerData playerData) {
    return playerData.viewFromEntityID(entityID) != null;
}

private boolean hasManagedEntity(int[] entityIDs, PlayerData playerData) {
    if (entityIDs == null) {
        return false;
    }
    for (int entityID : entityIDs) {
        if (isManagedEntity(entityID, playerData)) {
            return true;
        }
    }
    return false;
}
```
> Check first: `grep -n "isManagedEntity" core/.../PacketEntityViewController.java`. If upstream already defines either helper, keep upstream's and do not duplicate.

- [ ] **Step 4: Re-apply passenger/leash tolerance for unmanaged related entities** (port from `b574aac`)

In `handleEntityPassengers(int entityID, int[] passengers, ...)`, after the `entity == null` check and before the retry block, insert:
```java
            if (hasManagedEntity(passengers, playerData)) {
                return false;
            }
```
In `handleLeashEntity(...)`, after the `leashed == null` check and before the retry block, insert:
```java
            if (isManagedEntity(leashingEntity, playerData)) {
                return cancelIfEnabledAndHidden(leashingEntity, playerData);
            }
```
> If upstream's method signatures differ (e.g. no `retriesRemaining`), adapt the insertion point but keep the guard semantics: a null primary entity with a managed related entity must not error/spam.

- [ ] **Step 5: Compile core**

Run: `./gradlew :core:compileJava`
Expected: `core` compiles clean. (Callers in `packetevents` still broken — fixed in Task 4.)

- [ ] **Step 6: Commit**

```bash
git add core/
git commit -m "feat: gate entity-spawn culling to allowlisted targets (port onto upstream)"
```

---

## Task 4: Wire the filter into the PacketEvents entity controller

**Files:**
- Modify: `packetevents/.../viewcontrollers/PacketEventsEntityViewController.java`
- Modify: `platform-paper/.../paper/packets/PaperPacketEventsEntityViewController.java`

- [ ] **Step 1: Hold a `PacketEventsTargetFilter` in the controller**

Confirm the constructor accepts a filter (the cherry-picked code expects this). The field + null-safe default already exist in the fork version:
```java
private final PacketEventsTargetFilter targetFilter;
// in constructor:
this.targetFilter = targetFilter == null ? PacketEventsTargetFilter.DISABLED : targetFilter;
```
Re-add these to upstream's class if the cherry-pick didn't (upstream's constructor signature differs).

- [ ] **Step 2: Compute `shouldCullEntity` at the SPAWN_ENTITY seam and pass it through**

In the `SPAWN_ENTITY` case of `handleEntityPackets`, replace upstream's call:
```java
                WrapperPlayServerSpawnEntity packet = new WrapperPlayServerSpawnEntity(event);
                boolean isPlayer = packet.getEntityType().isInstanceOf(EntityTypes.PLAYER);
                boolean shouldCullEntity = targetFilter.shouldCullEntity(packet.getEntityType(), isPlayer);
                if (handleEntitySpawn(packet, packet.getEntityId(), isPlayer, shouldCullEntity, playerData, world, currentTick) == REQUIRE_EVENT_CANCELLATION)
                    event.setCancelled(true);
```
> Keep upstream's `entityID` argument (`packet.getEntityId()`) — that is the new param. The fork's version predates it; merge both.

- [ ] **Step 3: Pass the filter from the Paper subclass**

In `PaperPacketEventsEntityViewController`, accept a `PacketEventsTargetFilter` (built from `PaperTargetFilterService`) and forward it to `super(...)`. Match upstream's other constructor args.

- [ ] **Step 4: Compile**

Run: `./gradlew :packetevents:compileJava :platform-paper:compileJava`
Expected: compiles (block controller may still error until Task 5 — acceptable; entity path must be clean).

- [ ] **Step 5: Commit**

```bash
git add packetevents/ platform-paper/
git commit -m "feat: pass allowlist filter into PacketEvents entity controller"
```

---

## Task 5: Gate tile-entity culling + wire the block controller

**Files:**
- Modify: `packetevents/.../viewcontrollers/PacketEventsBlockViewController.java`
- Modify: `platform-paper/.../paper/packets/PaperPacketEventsBlockViewController.java`
- Modify: `platform-paper/.../paper/RaycastedAntiESP.java`

- [ ] **Step 1: Build the tile-entity allowlist (block-state IDs) from the filter service**

`PaperTargetFilterService` (cherry-picked) already expands `block-entities` + `block-entity-groups` into a set of block-state IDs. Confirm it exposes something like `Set<Integer> allowedTileEntityBlockIds()` (or port the fork's accessor). The block controller must only insert/track a tile entity when its block ID is in that set.

- [ ] **Step 2: Gate the insertion point** (port from `b574aac` block-view hunks)

In `PacketEventsBlockViewController`, at the point where a tile entity is inserted into the block view (the `insertTileEntity*` call sites), guard with the allowlist:
```java
        if (!targetFilter.shouldCullTileEntity(blockID)) {
            return; // pass through normally; do NOT mask to stone, do NOT track
        }
```
> Upstream now uses `insertTileEntityIfAbsent` / `updateOrInsertTileEntity` (mutable tile entities). Apply the guard BEFORE those calls. This is the key fix: un-allowlisted block entities (signs, decorated pots, heads) are never tracked, so they can never get stranded as stone.

- [ ] **Step 3: Inject the filter into the Paper block controller + construct it on enable**

In `PaperPacketEventsBlockViewController`, accept + forward the filter. In `RaycastedAntiESP` (plugin `onEnable`), build `PaperTargetFilterService` from config and pass a `PacketEventsTargetFilter` view of it into both Paper controllers. Match upstream's wiring/order.

- [ ] **Step 4: Full compile**

Run: `./gradlew shadowJar`
Expected: BUILD SUCCESSFUL; jar produced.

- [ ] **Step 5: Commit**

```bash
git add packetevents/ platform-paper/
git commit -m "feat: gate tile-entity culling to allowlisted block entities"
```

---

## Task 5b: Defensive SHOW-recovery (belt-and-suspenders)

**Files:**
- Modify: `packetevents/.../viewcontrollers/PacketEventsBlockViewController.java` (`processTileEntityTransitions`)
- Modify: `packetevents/.../viewcontrollers/PacketEventsEntityViewController.java` (`processEntityTransitions`)

Rationale: even after the upstream race fix, a SHOW that finds no tracked state must NOT leave the client looking at stone / a hole. Recover instead of silently `continue`.

- [ ] **Step 1: Tile-entity SHOW recovery**

In `processTileEntityTransitions`, replace the `state == null || blockID()==0` silent skip:
```java
                case SHOW -> {
                    if (state == null || state.blockID() == 0) {
                        // Tracking was lost before reveal — do NOT leave the client on the hidden (stone) block.
                        // Send the server's real current block so the client re-syncs from authoritative state.
                        int realId = resolveRealBlockId(location); // from blockInfoResolver / world snapshot
                        if (realId > 0) {
                            viewer.writePacketSilently(new WrapperPlayServerBlockChange(
                                new Vector3i(location.blockX(), location.blockY(), location.blockZ()), realId));
                        }
                        Logger.warning("Tile SHOW with missing state — recovered real block. loc=" + location, 2, PacketEventsBlockViewController.class);
                        continue;
                    }
                    ...existing reveal...
                }
```
> If no `resolveRealBlockId(location)` accessor exists, add a thin one on `blockInfoResolver`/the Paper resolver that reads the current block-state id at that location. Keep it minimal.

- [ ] **Step 2: Entity SHOW recovery**

In `processEntityTransitions`, the `entity == null` branch already logs and `continue`s; that is acceptable (a destroyed entity cannot be re-spawned without its cached data). Leave it, but downgrade the log to `info` level if it proves noisy after the race fix. No code change required beyond the log level.

- [ ] **Step 3: Build + commit**

```bash
./gradlew shadowJar
git add packetevents/
git commit -m "fix: recover real block on tile SHOW with missing state (no permanent stone)"
```
Expected: BUILD SUCCESSFUL.

---

## Task 6: Restore the full allowlist (re-hide holograms/signs — now they reveal)

**Files:**
- Modify: `platform-paper/src/main/resources/config.yml`

Goal per the user: keep hiding this stuff, just have it reveal correctly. With the reveal fixed (Tasks 0/5/5b), restore the original fork allowlist so holograms, signs, banners, heads, etc. are hidden out of LOS but reveal point-blank / on LOS.

- [ ] **Step 1: Restore the full `target-filter` lists** (original fork values — verify against `git show 6d1da4d:platform-paper/src/main/resources/config.yml` if unsure)
```yaml
    entities:
        - minecraft:villager
        - minecraft:copper_golem
        - minecraft:armadillo
        - minecraft:wolf
        - minecraft:cat
        - minecraft:ocelot
        - minecraft:allay
        - minecraft:bee
        - minecraft:iron_golem
        - minecraft:snow_golem
        - minecraft:item_frame
        - minecraft:glow_item_frame
        - minecraft:armor_stand
        - minecraft:painting
    block-entities:
        - minecraft:campfire
        - minecraft:soul_campfire
        - minecraft:decorated_pot
        - minecraft:bell
        - minecraft:jukebox
        - minecraft:conduit
        - minecraft:beacon
    block-entity-groups:
        - shulker_boxes
        - signs
        - hanging_signs
        - banners
        - wall_banners
        - beds
        - heads_and_skulls
```
> `moving_piston` is intentionally omitted (see Task 7 Step 3 — likely the cause of stone'd redstone, not a base-leak vector). Add it back only if Task 7 decides to keep it.

- [ ] **Step 2: Build + commit**

```bash
./gradlew shadowJar
git add platform-paper/src/main/resources/config.yml
git commit -m "config: restore full allowlist (reveal now fixed, safe to re-hide)"
```
Expected: BUILD SUCCESSFUL.

---

## Task 7: Port the remaining fork extras (evaluate each)

**Files:** as touched by each commit.

- [ ] **Step 1: Diagnostics command** — cherry-pick `83bc414` (`/raesp stats|trace|benchmark`). Resolve conflicts against upstream's command class. Build + commit.

- [ ] **Step 2: Visibility tracing** — from `bb9a4e8`, port the `VisibilityTraceService` + the `/raesp trace` recording calls. **Re-evaluate the "reveal fix" hunks**: upstream already reworked reveal/eviction (`5ba3140`, spawn-reconciliation commits). Apply a fork reveal hunk ONLY if the behavior it fixes is still missing upstream — otherwise drop it (it was a symptom patch for a bug upstream fixed at the source). Build + commit.

- [ ] **Step 3: Moving-piston filter** (`2335427`) — port only if pistons are a genuine ESP leak vector for this server. Default: **drop it** (pistons aren't base markers; this was a likely source of stone'd redstone). Document the decision in the commit message.

- [ ] **Step 4: Unknown entity-sync + passenger logging** (`66cef7a`, `1b86e64`) — these are log-noise/robustness tweaks. Port if the underlying log spam still occurs against upstream; otherwise drop. Build + commit.

- [ ] **Step 5: Full build**

Run: `./gradlew shadowJar`
Expected: BUILD SUCCESSFUL.

---

## Task 8: Integration verification on an isolated server

**Files:** none (runtime verification)

- [ ] **Step 1: Launch the in-repo test server**

Run: `./gradlew runServer`
Expected: Paper starts, plugin loads, no exceptions referencing `target`, `View`, or `NullPointer` on enable. If load fails on API drift, flip the dev bundle to `26.1.2.build.+` (see "Known gotcha"), rebuild, retry.

- [ ] **Step 2: Reproduce the exact user scenarios (full allowlist active)**

In-game (or via Paper MCP console): place a sign, a player head, an armor-stand hologram (DecentHolograms), and a beacon. With the full allowlist restored (Task 6), verify the two failure cases the user reported:
- **Standing on top of the hologram** → it is VISIBLE (point-blank, `total <= alwaysShowRadius`). Not gone.
- **Direct LOS of the sign** → it shows the REAL sign, never stone.
Then walk ~60 blocks away (past `raycast-radius`), turn around, walk back. Run `/raesp trace <self>` while doing it.
Expected: out of LOS → hidden/stone (anti-ESP working); point-blank or direct-LOS → revealed. The trace shows `TRANSITION_SHOW` reaching the client, NOT `show-skipped … missing-entity` and NOT a SHOW landing on a missing tile state.

- [ ] **Step 3: Hammer the reveal race**

Repeat the walk-away/return and look-away/look-back cycles ~10× on the hologram and the sign, ideally with 2+ players online (the race is per-player and load-sensitive). Watch console for any `missing-entity` (entity) or the new `Tile SHOW with missing state` (block) warnings.
Expected: every cycle reveals; zero stranded targets; `/raesp stats` shows `hidden*` counts return to 0 when everything is in view. Any residual warning = the upstream race fix didn't fully cover this path → return to systematic-debugging Phase 1 with the trace, do not patch blindly.

- [ ] **Step 4: Record evidence**

Capture `/raesp stats` output before/after and any console warnings into the PR description. If a target still strands hidden, STOP and return to systematic-debugging Phase 1 with the trace output — do not patch symptoms.

---

## Task 9: Re-establish upstream tracking ("back in the stream")

**Files:** none (git/repo workflow)

- [ ] **Step 1: Push the re-sync branch**

```bash
git push -u origin resync/upstream-allowlist
```

- [ ] **Step 2: Open the integration PR into the fork's `main`**

```bash
gh pr create --base main --head resync/upstream-allowlist \
  --title "Re-sync onto upstream + fix reveal-stranding (stone blocks / vanished holograms)" \
  --body "Rebuilds the allowlist feature on current Cubicake/RaycastedAntiESP main, inheriting upstream's thread-safe eviction + spawn-reconciliation + mutable tile-entity fixes. Trims allowlist to base-leakers only. See docs/superpowers/plans/2026-06-02-piecloak-upstream-resync.md."
```

- [ ] **Step 3: Set up clean ongoing tracking**

Document in the fork README (or the existing `f8c54e4` maintenance doc) the standing workflow: keep allowlist changes as a thin, regularly-rebased branch on top of `upstream/main`, and periodically `git fetch upstream && git rebase upstream/main` instead of letting the fork drift 46 commits again. Consider proposing the allowlist feature upstream as a PR so it stops being a fork at all.

- [ ] **Step 4: Clean up the worktree after merge**

```bash
cd /d/BadgersMC-Dev/PieCloak
git worktree remove ../PieCloak-resync
```

---

## Self-Review

- **Spec coverage:** Re-sync onto upstream → Tasks 0,1,9. Reveal/stone bug fix → inherited via upstream (Task 0 base) + tile-entity gate (Task 5). Hologram fix → Tasks 3,4,6 (armor_stand no longer culled). Both-together goal → whole plan. ✔
- **Placeholder scan:** No "TBD"/"handle edge cases". Conditional steps (helper may exist upstream; signatures may differ) give explicit grep checks + adaptation rules rather than hand-waving. ✔
- **Type consistency:** `shouldCullEntity` threaded identically through `handleEntitySpawn`/`handleEntitySpawn0` (Task 3) and the call site (Task 4). `targetFilter.shouldCullEntity(EntityType, boolean)` matches the cherry-picked `PacketEventsTargetFilter` API. `shouldCullTileEntity(int)` (Task 5) is asserted to exist on the filter — **verify against the cherry-picked `PacketEventsTargetFilter` in Task 1 Step 2; if the method is named differently, use that name throughout Task 5.** ✔
- **Risk:** Tasks 3–5 are the real work and depend on upstream method shapes that may have shifted since this plan was written — always re-read the upstream method before editing, port semantics not line-for-line.
