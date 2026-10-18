# KubeJS for Minecraft 26.3 / NeoForge 26.3.0.89 (unofficial port)

This branch (`26.3`) is an **unofficial** port of KubeJS to Minecraft 26.3, built for the Zoompack 5 modpack. It is
not released or endorsed by the KubeJS team. KubeJS is © latvian.dev and contributors, licensed under the
GNU LGPL v3 (`LICENSE.txt`); this port keeps that licence, and every change here is LGPL v3 too.

Version: `26.3-8.1.0-port.1`. Rhino dependency: `2603.2.8-port.1` (see `../Rhino/PORTING-26.3.md`).

## Where it comes from

| Ref | What it is |
|---|---|
| `origin/2603` (`ada579b4`) | Upstream's 26.3 branch. Still a 26.1.2 placeholder (gradle.properties says 26.1.2), no port commits. |
| `origin/2601` (`6f0827bc`) | Upstream 26.1.2, KubeJS 8.0.7. |
| `wolfie/2603` (`4812f95c`) | [KubeJS-Mods/KubeJS#1189](https://github.com/KubeJS-Mods/KubeJS/pull/1189) "26.1 -> 26.3 Port" by **wolfieboy09**, open against `2603`, built on NeoForge 26.3.0.76. Includes upstream `2601` up to `8c703ed0` (one commit behind: "bump version"). |
| `26.3` (this branch) | `wolfie/2603` + the changes below. |

Almost all of the 26.1 -> 26.3 migration (about 120 files) is wolfieboy09's work in #1189; its description is the
best changelog for it (loot number providers, worldgen registry renames, advancement predicate move, recipe
loading, condition context, SDL input, `Minecraft#gui`, block fall callbacks, data component removals and so on).
This branch adds what was needed to make it build on NeoForge 26.3.0.89 and actually work at runtime.

## Changes on top of #1189

1. **NeoForge 26.3.0.89** (`gradle.properties`: `neoforgeVersion`, `neoforgeVersions=[26.3.0.89,)`). ModDevGradle 2.0.147
   and Gradle 9.8.1 from the PR work unchanged.
2. **`DataPackRegistriesHooks#getReloadableRegistries` is gone** after 26.3.0.76 (as is `getDataPackRegistries`); the
   vanilla `RegistryDataLoader.WORLD_REGISTRIES` / `RELOADABLE_REGISTRIES` lists now hold the modded entries.
   `ServerScriptManager` iterates both. The PR had switched it from world registries to reloadable ones only, which
   would have stopped `ServerEvents.registry('damage_type' | 'worldgen/...', ...)` from working.
3. **`ServerEvents.recipes` never fired in #1189.** 26.3 loads recipes as a reloadable registry (in
   `ReloadableServerRegistries`, before `ReloadableServerResources`/`RecipeManager` exist) and NeoForge removed
   `ModifyRecipeJsonsEvent`, which is what used to call `RecipesKubeEvent#post`. The PR removed the handler and
   nothing replaced it. New:
   - `core/mixin/ResourceManagerRegistryLoadTaskMixin` wraps the `ParallelMapTransform.schedule` call in
     `ResourceManagerRegistryLoadTask#load` (lambda `lambda$load$1`) for `Registries.RECIPE` only: it hands the
     listed recipe files to `recipe/RecipeResourceLoader`, which reads them as JSON, runs the pre-event steps that
     used to live in `RecipeManagerMixin#prepare` (bind pending/script tags, recipe schema events,
     `ServerEvents.specialRecipeSerializers`), posts the recipe event inside the `RecipesKubeEvent.INSTANCE` scope,
     and returns the modified JSON as in-memory `Resource`s (keeping each original pack as the source; added recipes
     use KubeJS's internal virtual pack). Files that are not valid JSON are passed on untouched.
   - The now-dead `RecipeManagerMixin#kjs$preRecipeLoad` was removed.
   - Recipes are parsed on loader worker threads, where the scoped value is not bound, so `PendingRegistrationMixin`
     falls back to the new `RecipesKubeEvent.PARSING` to report recipes that failed because of a script.
4. **Default data components are built early.** 26.3 runs `DataComponentInitializers#build` after the reloadable
   registries, so during the first load no item has components and every `ItemStack` the recipe event creates threw
   `Components not bound yet` (each of the 2,042 vanilla recipes fell back to "unknown", and `event.shaped(...)`
   failed). `RecipeResourceLoader` builds and applies them from the context registries first; vanilla builds and
   applies them again afterwards. 26.1 KubeJS also applied pending components early (the "TODO: i would like to be
   able to live without these two calls").
5. **Removed recipes that something still references.** Since 26.3, advancements are a reloadable registry and
   `recipe_unlocked` / `recipe_crafted` triggers hold registry references to recipes, so removing e.g.
   `minecraft:stick` left an unbound holder and the reload failed with `Unbound values in registry minecraft:recipe`.
   `core/mixin/RegistryLoadTaskMixin` (HEAD of `freezeRegistry`, recipe registry only) binds just those ids to a new
   inert `recipe/RemovedRecipe` (`kubejs:removed` recipe type and serializer; never matches, `isSpecial`, not in the
   recipe book). Removed recipes nothing refers to are still absent. The serializer and type are registered with
   the other KubeJS recipe serializers (not in `serverOnly` mode; there the type is unregistered but still works).
6. **Better Advanced Tooltips is a required dependency** (`neoforge.mods.toml`, both sides). `TextIcons`,
   `KubeJSCommands` and `ChangesForChat` load BAT classes on the server, so without it the game crashes with
   `ClassNotFoundException: BATIcons` (the toml said optional/client; Modrinth already lists it as required).
   Official 26.3 BAT exists: `dev.latvian.mods:better-advanced-tooltips:2603.1.1-build.13` (MIT).
7. `KubeJS.MC_VERSION_NUMBER`/`MC_VERSION_STRING` = `2603`/`"26.3"` (scripts see them as `Platform.mcVersion` etc.;
   they were still 2601), support URLs say `mc=2603`, version suffix `-port.1`.
8. Rhino `2603.2.8-port.1` instead of `2101.2.8-build.91` (the old one also loads; see the Rhino notes).
9. EMI integration re-enabled against SEMI (see below).

## Dependencies

| Dependency | Kind | 26.3 source |
|---|---|---|
| NeoForge 26.3.0.89+ | required | - |
| Rhino | required | this port's `../Rhino` (`2603.2.8-port.1`) |
| Better Advanced Tooltips | required (de facto) | official `2603.1.1-build.13` |
| tiny-java-server 1.0.0-build.45, animated-gif-lib 1.7 | bundled (jarJar), plain Java | unchanged |

## Optional integrations

| Integration | State |
|---|---|
| JEI | Compiled against JEI API 31.7.0.47 (26.3). Not runtime-tested (client only). JEI 31.15.0.79 is on Modrinth for 26.3. |
| REI | Commented out upstream since 26.1. REI 26.3.824 now exists for NeoForge, so it could be re-enabled. |
| EMI | **Re-enabled, client-tested** against [SEMI](https://github.com/Pandaismyname1/SEMI) (Seasonal EMI, an EMI port to 26.3 that keeps EMI's API and the `emi` mod id): `compileOnly dev.emi:semi-neoforge:1.1.24-SNAPSHOT+26.3:api` from SEMI's `maven` branch; optional range `[1.1.24-SNAPSHOT,)` (fixes the old `[{emiVersion},)` typo). The integration sources were commented out upstream since 26.1 and compiled unchanged apart from `ResourceLocation` -> `Identifier`. With EMI present the JEI plugin disables itself, as before. A server with SEMI passes the GameTests, and in the client test the plugin ran and applied `removeEntries`. |
| Architectury | Integration commented out upstream ("not updated to 26.1 yet"); the plugin entry logs "required mod 'architectury' not loaded, skipping". Architectury API 22.0.5 exists for 26.3. |
| GameStages | Compiled, only active with the mod present; untested. |

## Script-facing differences from KubeJS 8.0.x on 26.1

From #1189 (details in its description):
- `BlockBuilder.bounciness(...)` -> `restitution(float | callback)`; `afterFallenOn(...)` removed
  (`AfterEntityFallenOnBlockCallback` -> `BounceRestitutionCallback`); `fallenOn(...)` unchanged.
- Binding `GLFWInput` -> `SDLInput` (LWJGL uses SDL now).
- Loot number providers: `NumberProvider` is gone; wrappers for `ContextIntProvider`/`ContextFloatProvider`
  (`{min, max}` with an int min and float max is no longer expressible; `{n, p}` binomial is int-only).
- Item `mapColor` setter is a no-op (component removed by Mojang).
- `KubeDataGenerator` `setCompostable`/`removeCompostable`/`setFurnaceFuel`/`removeFurnaceFuel` removed; use the
  ItemBuilder compostable / cooking-fuel options (now data components).
- Keybind category lang key `key.categories.kubejs` -> `key.category.kubejs.kubejs`.
- Shovel/axe/hoe item types are built through `Item.Properties` (`DiggerItemBuilder` handles it).

From this port / vanilla 26.3:
- `Platform.mcVersion` is `2603` / `"26.3"`.
- Removing a recipe that an advancement (or another mod) refers to leaves a `kubejs:removed` placeholder under that
  id: `recipeManager.byKey(id)` is present but never matches and is hidden. Removed recipes nothing refers to are gone.
- `ServerEvents.registry` now also targets reloadable registries (loot tables, predicates, item modifiers,
  advancements, recipes...), in addition to world registries. Only `damage_type` was tested.
- Vanilla renames scripts hit directly (not KubeJS bugs): `Entity#getTags()` is `entityTags()`, so the old
  `entity.tags` bean is gone (use `entity.addTag/removeTag/entityTags()`); dyes are one item (`Items.DYE.<color>()`),
  so ids like `minecraft:purple_dye` are gone; `EntityType` constants moved to `EntityTypes`.
- NeoForge renames for `NativeEvents` users: `BlockEvent$BreakEvent` is gone (use
  `net.neoforged.neoforge.event.level.block.BreakBlockEvent`), `ModConfig.Type.COMMON/SERVER` are `LOCAL/SYNCED`.
- Java plugins: `ModifyRecipeJsonsEvent` is gone; `KubeJSPlugin#beforeRecipeLoading` is still called.

## Verification

- `./gradlew build` succeeds (JDK 25, Gradle 9.8.1); access transformer validation passes. KubeJS has no unit tests.
- `../smoketest` is a small GameTest mod (outside this repo) that copies this jar, the Rhino jar, BAT and SEMI into
  `run/mods` and smoke scripts into `run/kubejs`, then runs `runGameTestServer` on NeoForge 26.3.0.89:
  - startup: `StartupEvents.registry('item'|'block')` -> `kubejs:smoke_item` (stack size 16), `kubejs:smoke_block`
    (hardness 1.5) and its block item exist;
  - server: `event.shaped` / `event.shapeless` added (4 dirt crafts `kubejs:smoke_item`), `remove({id: 'minecraft:stick'})`
    and `remove({output: 'minecraft:torch'})` (2 planks no longer craft), item and block tags added/removed,
    `ServerEvents.registry('damage_type')` entry present, `PlayerEvents.loggedIn` ran for a mock player, and
    `/reload` re-runs all of it;
  - 0 startup and 0 server script errors (read from `ConsoleJS`). Result: **All 8 required tests passed**.
  - Every common mixin is `required` with `defaultRequire: 1`, so they all applied (a failed injection would abort).
- `../smoketest` `runClient` (opens a window, quits itself; `-Dkjs_smoke.client=true`): dismisses the mod-loading
  warnings screen (only SEMI's `@OnlyIn` notice), creates a fresh creative test world, then checks after 200 ticks:
  - the `ItemEvents.modifyTooltips` line from `client_scripts` is on `kubejs:smoke_item`'s tooltip;
  - 0 client script errors;
  - EMI ran the KubeJS plugin ("Initialized plugin from kubejs") and `RecipeViewerEvents.removeEntries('item')` hid
    `minecraft:dirt` from EMI's index (2,004 stacks, stone still present).
  Result: **SMOKE-CLIENT: ALL PASSED**. All client mixins loaded (they are `required` too).
- **Not verified:** the JEI plugin (JEI not installed in the test), key bindings / SDL input, the error screen, datagen,
  the web server, dedicated-server + remote client sync, and a full modpack boot.

## Known risks / follow-ups

- The early `DataComponentInitializers#build` uses the previous registry layers plus this reload's tags. Item
  components that depend on reloadable registries (none in vanilla) would see the previous reload's values during the
  recipe event only.
- `RecipesKubeEvent.PARSING` is a static: two servers reloading at once in one JVM would share it (`failedCount` is not
  atomic either). Fine for one server per JVM.
- The mixin targets the synthetic lambda `lambda$load$1`; a NeoForge/vanilla change to `ResourceManagerRegistryLoadTask`
  will make it fail loudly at startup (required injection), not silently.
- Upstream may solve recipe loading differently (e.g. a NeoForge hook); prefer theirs when it lands.

## Building

```
JAVA_HOME=<JDK 25> ./gradlew build          # needs ../Rhino published first: (cd ../Rhino && ./gradlew publishToMavenLocal -Pbuild_tag=port.1)
cd ../smoketest && ./gradlew runGameTestServer
```

## Keeping it rebased

Remotes: `origin` = KubeJS-Mods/KubeJS, `wolfie` = wolfieboy09/kubejs (PR #1189's head branch `2603`).
The port is on `26.3` on top of `wolfie/2603`, published at [zoom-packers/kubejs](https://github.com/zoom-packers/kubejs/tree/26.3) (remote `zp`). Jars are released on [zoom-packers/zp5-mods](https://github.com/zoom-packers/zp5-mods/releases/tag/kubejs-26.3-8.1.0-port.1). To update:

```
git fetch origin wolfie
git rebase wolfie/2603        # while #1189 is open
git rebase origin/2603        # once #1189 is merged and upstream's 2603 has the port
```

Conflicts will mostly be in `ServerScriptManager`, `RecipeManagerMixin`, `PendingRegistrationMixin` and
`kubejs.mixins.json`. If upstream ships its own recipe-loading fix, drop items 3-5 above. Once an official 26.3
KubeJS is on Modrinth/maven.latvian.dev, switch the pack to it and retire this branch.

## Contributing upstream

Items 2-7 are candidates for review comments or a follow-up PR on #1189 (the user decides whether to send them).
