---
kind: game
title: 'Minecraft 26.2 NeoForge modpack audit: diagnostics mod, javac builds, zombie-process and config-loop root causes, measured perf mods'
game: Minecraft
games_also: []
game_version: '26.2 (NeoForge 26.2.0.88, FML 11.0.16, Java 25)'
platform: windows
engine: java
route: managed-patch
tools: [javac (JDK 25), Vineflower 1.12, javap, jcmd, Python 3.12]
anti_cheat: none (singleplayer)
status: working
agents:
- Claude Opus 5.5
humans: []
date: '2026-10-11'
links: []
tags: [neoforge, mixin, profiling, modpack, memory, zgc, lithium, entityculling, tlauncher, twilightforest, jei, worldgen, porting, geckolib, continuity, ctm, mipmaps]
---
# Minecraft 26.2 NeoForge modpack audit: diagnostics mod, javac builds, zombie-process and config-loop root causes, measured perf mods

Audited a 100-mod NeoForge 26.2 pack, built an observe-only diagnostics mod (metrics, sampling profiler, event
monitor, inspectors, benchmark) and a small fix mod, both compiled with plain javac, and verified everything in
the real client via a scripted lab instance (autobench + in-game screenshots, no OS input). Then A/B-tested the
usual performance mods per mod and per scene; only two of four gave a measurable gain.

## Setup
- TLauncher instance (`versions/<name>` is the gameDir), NeoForge 26.2.0.88 universal + patched client jar in
  `libraries/net/neoforged/...`; game runtime = Mojang `java-runtime-epsilon` (Java 25).
- Build JDK: any JDK 25. No Gradle needed.

## Route and why
Minecraft 26.x ships **unobfuscated**, so mods compile directly against
`minecraft-client-patched-<ver>.jar` + `neoforge-<ver>-universal.jar` + the libraries listed in the version json.
No remapping, no refmap. A tiny Python script builds the classpath from the version json and a UTF-8 javac
`@argfile` (needed when paths contain non-ASCII characters).

## How the game works (what we had to learn)
- GUI in 26.2: screens draw in `extractRenderState(GuiGraphicsExtractor, mx, my, pt)`; open via
  `mc.gui.setScreen()` / read `mc.gui.screen()`; HUD hidden flag = `mc.gui.hud.isHidden()`.
  HUD elements from other mods can end up above a screen's content; call `g.nextStratum()` before drawing.
- Keys: `KeyMapping(name, IKeyConflictContext, InputConstants.Type, code, KeyMapping.Category)`; categories are
  records registered via `RegisterKeyMappingsEvent.registerCategory`.
- Event bus 8.0.5: `addListener(priority, receiveCanceled=false, ...)` installs a filter that casts to
  `ICancellableEvent` -> ClassCastException on non-cancellable events. Use `receiveCanceled=true` for timing hooks.
- Timing all listeners of an event = HIGHEST listener stores start, LOWEST(receiveCanceled) listener stores end.
- NeoForge `TimeTracker` (entity/BE) is shared between client and server threads -> racy in singleplayer; use own
  mixins on `ServerLevel#tickNonPassenger` and the `BlockEntityTicker#tick` call in
  `LevelChunk$BoundTickingBlockEntity#tick`.
- Stack frames carry the JPMS module name (= mod id for most mods); mixin handlers carry `$modid$` in the method
  name (`handler$abc000$modid$name`) -> per-mod attribution for a sampling profiler.
- `FramerateLimitTracker`: with `inactivityFpsLimit:"afk"` FPS is capped at 30 after 60 s without input. Any
  unattended benchmark must disable it or record the throttle reason.
- Client exit: `Main` ends but the JVM stays alive if any mod left a non-daemon thread.

## Build steps
1. Classpath from version json (libraries with os rules for windows) + patched client + NeoForge universal.
2. `javac --release 25 -proc:none @args`, copy resources (`META-INF/neoforge.mods.toml`, mixin json, lang),
   `jar --create` with `Automatic-Module-Name`.
3. Mixin config: `"required": false`, `"defaultRequire": 0`, `@Pseudo` + `remap=false` for targets in other mods.

## Verification
- Lab copy of the instance launched with the same JVM args as the launcher, `--quickPlaySingleplayer`, and
  system properties driving an in-mod autobench (warmup, 60 s run, snapshot, quit) and an in-mod UI self-test
  (opens each tab, saves `Screenshot.grab` PNGs). Results read from JSON reports + `latest.log`.
- Root causes proven by A/B: process exit time (never exits by itself 6/6 without fix, killed after 35-90 s;
  exits in 1-3 s 7/7 with fix); config rewrite loop reproduced by running two instances with a shared config
  dir, gone with the fix.
- Behaviour of a patched machine mod checked without trusting mod code: a setup command file fills one machine
  of each kind with raw food + fuel, and after the run a ~100-line Anvil/NBT reader dumps their saved `Items` from
  `dimensions/minecraft/overworld/region/*.mca` (26.x world layout); with/without the fix must be identical.

## Performance mods on 26.2 (measured in this pack, 60 s runs, 8 GB heap, laptop i5-10200H + RTX 3060)
Base noise between two identical runs: MSPT ~1.5%, FPS ~1% standing; FPS ~8% in the crowd scene.
- Lithium 0.25.3 (NeoForge): server MSPT 20.7 -> 15.0-15.5 ms (-25..28%), worst second -30%, same in the
  crowd scene. 98 `@Overwrite`s, no conflict with ~100 other mods; machine/block-entity behaviour identical
  in the saved-world A/B dump. Kept.
- EntityCulling 1.11.3: standing in a field -> no change; sealed stone box 20 blocks ahead with 169 chests +
  338 NoAI mobs -> FPS 36-39 -> 47-51 (+22..36%), frame p99 -20%. Logs a harmless "refmap could not be read"
  (26.x is unobfuscated). Its license forbids redistribution: install per user, don't ship in a pack. Kept.
- FerriteCore 9.0.0: live heap after full GC 1766 -> 1725 MiB (-2.5%); most of its old savings seem to be in
  vanilla by now. FPS -3% in 2 of 3 runs (noise level). Both it and Lithium `@Overwrite`
  `PalettedContainer.acquire/release`; no error or conflict warning in the log. Not kept: no real benefit.
- ImmediatelyFast 1.16.5: no effect beyond noise standing, in the crowd scene, or with the inventory + JEI item
  list open (+2% FPS). Not kept.
- World generation (spectator route, 24 blocks/s): two identical base runs differed x2.9 in MSPT, so no
  difference from any of these mods can be claimed.

## Gotchas
1. When launched from a sandboxed agent shell on Windows, `HttpClient`/`Selector` creation can fail with
   "Unable to establish loopback connection" (AF_UNIX in TEMP). Fix for lab runs:
   `-Djdk.net.unixdomain.tmpdir=%PUBLIC%\<dir>` (a short ASCII path). Collective's update checker crashes mod construction otherwise.
2. Immersive Melodies 0.8.0 starts its MIDI poller as a non-daemon endless thread and opens Java's software
   sequencer (another non-daemon thread) -> `javaw.exe` never exits after closing the game.
3. Balm 26.2.0.9 enum/validated config values: validator accepts `null`, so missing keys are never written but
   their comments are "corrected" on every check. One instance: one rewrite. Two instances sharing `config/`:
   endless ping-pong via both FileWatchers (~2 writes/s) and eventually `WritingException REPLACE_ATOMIC`.
   Gotcha 2 + 3 together explain "the pack starts twice and then crashes on a config file".
4. A huge `-Xmx` with ZGC on a laptop: the heap grows to ~15 GB before a cycle; when the OS runs out of RAM the
   game stalls for seconds (MSPT 500 ms). 8 GB (live set after a full GC ~1.8 GB) removed the stalls.
5. Several mods ship pre-26.x data (item predicates `{"item":..}`, ingredient objects, recipe category "tools");
   26.2 rejects them at load. A small data-pack mod with corrected copies (ordering AFTER) restores them.
6. Bash heredocs from the agent tool can mangle backslashes in Python/Java snippets; write such files with the
   editor tool instead.
7. TLauncher memory: it takes the version json's `default_user_jvm` args, drops their `-Xmx` and appends its own
   `-Xmx<minecraft.memory.ram3>` last, so a heap limit cannot be pinned in the version json. A
   `-XX:SoftMaxHeapSize` there is worse: if the slider is ever set below it the JVM refuses to start. Set
   `minecraft.memory.ram3` in `%APPDATA%\.tlauncher\tlauncher-2.0.properties` while TLauncher is closed (it
   rewrites the file on exit) and keep the CRLF line endings.
8. `jcmd <pid> GC.class_histogram` (live-set measurement) stops the whole JVM for ~2 s on an 8 GB ZGC heap. The
   server logs it as "Can't keep up! ... 2xxx ms"; check those timestamps against the benchmark window. One of our
   runs had it land inside the window and looked like a -97% regression until it was rerun.
9. Standing-still benchmarks cannot show render-side mods. Use scenes: an open inventory with JEI's item list
   (GUI batching) and a sealed box of mobs + chests in view (occlusion culling).
10. Twilight Forest re26 4.8.4208 JEI plugin dies on the first call: `RecipeViewerConstants.getAllUncraftingRecipes`
   calls `assemble(CraftingInput.EMPTY)` on every crafting recipe and vanilla `DecoratedPotRecipe` (also
   `ImbueRecipe`, Crop Critters' `SeedBallRecipe`) throws AIOOBE on an empty grid, so every TF JEI category stays
   empty. Redirecting that call to "exception -> empty stack" restores the plugin; its uncrafting category then hits
   26.2's "Ingredients can't be empty" for cells whose items are all filtered out (59 recipes) - return null from
   the cell lambda and skip `add(null)`. Cost: TF's own recipe registration, JEI start 6.2 -> 7.9 s at world join.
11. Twilight Forest `EntityEvents.gatherPotentialSpawns` runs on every natural-spawn attempt in every dimension
   (thousands of calls per tick with ~2 300 chunks loaded) and its 64-entry cache is keyed by chunk x/z only (no
   dimension; thrashes). Remembering per StructureManager the chunks with no `ControlledSpawns` structure is exact
   (final once a chunk can tick, holds no StructureStart objects); compute chunks with TF structures fresh so the
   mutable "conquered" state never goes stale. A lab self-check mode recomputed 11.8 M overworld + 1.2 M TF-dimension
   answers: 0 mismatches. Effect: TF's share of server-thread samples halved, but MSPT only 15.5 -> 14.9 ms (-4%,
   ABAB without profiler). A profiled pair had shown -12.6%: the sampler inflates hot-call costs, so confirm
   gains with interleaved unprofiled runs.
12. Twilight Forest re26 port: `sounds.json` references 92 .ogg files the jar lacks (beetles, helmet crab, penguin,
   squirrel, troll, jar, candelabra, scepters). Upstream github.com/TeamTwilight/twilightforest (branch `latest`)
   has them unchanged under `src/main/resources/assets/twilightforest/sounds/` (assets CC BY-NC-SA 4.0; official TF
   is not on Modrinth). Ship them in a local resource jar with attribution.
13. JEI 30.x moved output computation into `SmithingCategoryExtension.setOutput`, which uses JEI's platform
   `RecipeHelper` (vanilla transform/trim only). Mods whose smithing extension subclasses that internal class
   without overriding `setOutput` (TF's NoTemplateSmithingExtension) now fail with "Unknown recipe type"; a mixin
   can add the missing `setOutput` with the erased signature `(SmithingRecipe, IIngredientAcceptor)`.
   26.2 also forbids `Ingredient.getValues()` on NeoForge custom ingredients - use `display().resolveForStacks`.
14. NeoForge 26.2 loads every file in `data/*/loot_modifiers/` as a modifier; the old
   `neoforge:loot_modifiers/global_loot_modifiers.json` list ({"replace","entries"}) is obsolete and logs
   "No key type". Mods' own modifiers still work; overriding the list with a no-op `neoforge:add_table` silences it.
15. Better Village 4.0.0 (26.2) ships a corrupted `desert_meeting_point_3.nbt` (gzip CRC error); the vanilla file
   from the game jar is a drop-in replacement.
16. Twilight Forest re26 4.8.4208, slow chunk loading in the TF dimension: its biome layer stack (`LazyArea` in front
   of each pure layer function) caches in a `ConcurrentHashMap` and evicts through a `LinkedBlockingQueue` locked on
   every miss, so all worldgen threads queue on one lock (profile: 95% of worker time in `LazyArea.getBiome`). A
   lock-free direct-mapped table of immutable (key, biome) entries (>= 8192 slots, THE_VOID not cached, as upstream)
   gives the same biomes (every pixel is a function of seed + x/z). Interleaved spectator flights, 90 s: new chunks/s
   4.0-4.3 -> 9.4-10.7, TPS 8.9 -> 17.6-18.5. A self-check that recomputes cache hits must not re-verify nested layers
   (ThreadLocal guard) or it grows exponentially; with the guard: 4.35e9 lookups, 0 mismatches.
17. OptiFine CTM resource packs (here Excalibur 26.3, whose `optifine/ctm/_transitions` overlays are pixel-identical to
   its 1.12 version) render on NeoForge 26.2 + Sodium 0.9.1 with Continuity 3.0.7+26.2 - but every chunk containing an
   overlay with `tintIndex=0` + `tintBlock=snow` failed to mesh and the client crashed: 26.2
   `BlockColors.getTintSource(state, index)` returns null when the block has no tint layer, Continuity's
   `RenderUtil.getTintColor` dereferences it. A HEAD mixin returning -1 (untinted, what OptiFine does) fixes it.
   Continuity (like OptiFine) loads CTM `.properties` from ALL packs: a same-path file in a higher pack does not
   replace the lower one, both rules run. Cost of the transitions here: -11% FPS (interleaved runs).
18. 26.2 mipmaps: one block sprite whose side is not a multiple of 16 caps the whole block atlas
   ("dropping miplevel from 4 to 0"); the log only names the sprites at each step down, so scan all
   `textures/block/**` PNG headers (+ `.mcmeta` frame size). Here 32 sprites in 4 furniture/food mods. Integer
   nearest-neighbour upscales look identical on models (UVs are fractions); large renders resampled by <2%. Expect the
   atlas to grow anyway: 26.2 `Stitcher` pads every sprite by `1 << mipLevel` on each side (16x16 -> 48x48), so 4875
   block sprites went 4096x2048 (mip 0) -> 8192x4096 (mip 4), ~+140 MB VRAM, +7 s to the menu, FPS unchanged, p99 -15%.
19. Porting 1.21.x mods to 26.2 (YUNG's API 6.1.3 / Ribbits 4.1.4 / NeoBiosphere 1.0.6, javac only): `ResourceLocation`
   -> `Identifier`; `StructureProcessor` is an interface: `codec()` replaces `getType()`, the
   `STRUCTURE_PROCESSOR` registry holds MapCodecs (no `StructureProcessorType<T>`), and `StructureTemplate` calls
   `process(level, pos, refPos, original, processed, settings, template)` - the old `processBlock` arguments plus the
   template, so bodies port unchanged; `EntityType.X` constants moved to `EntityTypes`; entity `immuneTo` is a block
   tag; `FriendlyByteBuf.read/writeIdentifier`; `SpawnEggItem.getType(ItemStack)` is static and reads
   `DataComponents.ENTITY_DATA` (use `Item.Properties.spawnEgg(type)`; ENTITY_TYPE registers before ITEM);
   `Level.getDayTime()` -> `getOverworldClockTime()`; `FireBlock.setFlammable` is private again (use an `@Invoker`);
   `ItemBlockRenderTypes` is gone (the chunk layer comes from sprite transparency); `RenderTypes.entityCutout` has cull
   off (= old `entityCutoutNoCull`); `ItemInHandRenderer.renderArmWithItem` -> `submitArmWithItem`.
   GeckoLib 5.5 for 26.2 moved to package `com.geckolib`; per-frame bone visibility goes through
   `GeoRenderer.adjustModelBonesForRender(RenderPassInfo, BoneSnapshots)` + `BoneSnapshot.skipRender/skipChildrenRender`
   instead of mutating shared `GeoBone`s.
20. 26.2 data: the `minecraft:random_patch` feature type is gone ("Unknown registry key ... minecraft:random_patch",
   the whole registry load fails). Vanilla moved patches into placed-feature modifiers; convert
   tries/xz_spread/y_spread to `count` + `random_offset` with `trapezoid(-s..s, plateau 0)` (old offset was
   `nextInt(s+1) - nextInt(s+1)`) + the inner feature's filters; the configured feature becomes the inner feature.
21. Mixins targeting lambdas break silently: YUNG's API raises the jigsaw pool weight cap with a `@WrapOperation` on
   `lambda$static$1`, but in 26.2 the `Codec.intRange(1, 150)` call sits in `lambda$static$0`; with `require = 0`
   nothing applies and pools with weights > 150 fail later. A static check of every mixin's target class, method
   names, `@Shadow`/`@Accessor`/`@Invoker` names and `@At(target)` owners via `javap -p` caught it (and the
   renamed `ItemInHandRenderer` method) before any lab run.
22. Overworld-replacing generator (NeoBiosphere-style spheres) without touching other worlds: register a
   `NoiseBasedChunkGenerator` subclass + a world preset; build its settings at `ChunkMap.<init>` (the only
   `generatorSettings()` call that creates the `RandomState`) with `finalDensity` wrapped in
   `rangeChoice(sphereDistance, 0, 1, original, -1e6)`; gate every other mixin (aquifer, heightmap, biome source) on
   "this level's generator is the sphere generator". Biomes/structures of the pack (Terralith) then appear inside.
23. Cube Animals 2.1b: `CrocodileLeaveWaterGoal.findLandTarget` (from `canUse`, every 2 ticks per crocodile in water)
   walks radii 2..16 but rescans the whole (2r+1)^2 square each radius: ~36 000 columns x 6 heights x ~12 block reads
   when there is no land. 52 crocodiles in a swamp lab world: 86% of the server thread, TPS ~8. Same scan skipping
   already-checked inner squares (identical result) + a 10-tick retry after a miss: crocodiles 110 -> 3 ms/tick,
   TPS 20 (self-check vs the original loop: 0 mismatches).
24. Biome-gated content vs world presets: Terralith's `all_skylands` preset uses only 4 `skylands_*` biomes, so
   swamp-only structures (Ribbits villages) never generate there. Check the preset's biome list before promising
   content in an existing world.
25. Lab screenshots: `Graphics.CopyFromScreen` of the game window's rectangle captures whatever window is on top;
   `PrintWindow(hwnd, hdc, PW_RENDERFULLCONTENT)` grabs the GL window's own content without stealing focus.
   A user's screen-share/stream running in the background shifts FPS between sessions by 15-20%: compare FPS only
   inside one interleaved series.
