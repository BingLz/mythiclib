# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build and test commands

This is a multi-module Maven project. There is no Maven wrapper in the repository, so use a locally installed `mvn`.

- Compile/package the full project: `mvn clean package`
- Install the full project locally: `mvn clean install`
- Build the shaded distribution JAR: `mvn -pl mythiclib-dist -am clean package`
  - Output is configured as `target/MythicLib-${project.version}.jar` from `mythiclib-dist/pom.xml`.
- Compile the core plugin module and its dependencies: `mvn -pl mythiclib-plugin -am compile`
- Run all plugin tests: `mvn -pl mythiclib-plugin test`
- Run a single test class: `mvn -pl mythiclib-plugin -Dtest=AdventureParserTest test`
- Run a single test method: `mvn -pl mythiclib-plugin -Dtest=AdventureParserTest#testHexColorTag test`

Full builds include many `mythiclib-v...` NMS wrapper modules. Several require locally installed/remapped Spigot artifacts generated with BuildTools; see `README.md` for the BuildTools commands and Java-version caveats. If you only need a specific Minecraft version wrapper, the README explicitly notes that unused wrapper modules can be removed/omitted to save time.

## Project structure

- `pom.xml` is the parent POM for all modules. It sets Java source/target 11 and registers the plugin, distribution, RPG compatibility, and per-Minecraft-version wrapper modules.
- `mythiclib-plugin/` is the main Bukkit/Paper plugin module. Its entry point is `io.lumine.mythic.lib.MythicLib`, declared in `mythiclib-plugin/src/main/resources/plugin.yml`.
- `mythiclib-dist/` assembles the deployable plugin artifact by depending on `mythiclib-plugin`, `mythiclib-rpg`, and every version wrapper module, then shading/relocating bundled libraries such as Gson, HikariCP, Crunch, bStats, and AnvilGUI.
- `mythiclib-rpg/` contains integrations for RPG/class/level/mana providers such as mcMMO, Fabled, Heroes, AuraSkills/AureliumSkills, Skills, SkillsPro, and related plugins.
- `mythiclib-v1_14_r1/` through `mythiclib-v26_1_r0/` contain version-specific `VersionWrapper_*` implementations for Bukkit/Spigot/Paper/NMS API differences.

## Runtime architecture

`MythicLib` initializes global managers and hooks in `onLoad()`/`onEnable()`:

- Server version detection is handled by `version.ServerVersion`, which computes the CraftBukkit revision and reflectively loads a matching `version.wrapper.VersionWrapper_*`. If no exact wrapper exists, it falls back to the most recent wrapper in compatibility mode.
- Core feature managers are long-lived fields on `MythicLib`: damage, entities, stats, config, elements, skills, flags, fake events, damage indicators, regen indicators, mitigation, and on-hit effects.
- Optional plugin integrations are detected at startup with Bukkit's plugin manager. Hooks include WorldGuard/Residence flags, MythicMobs, PlaceholderAPI, ProtocolLib/PacketEvents, Spartan, mcMMO, Fabled, RealDualWield/DualWield, hologram providers, and RPG providers.
- Reload order matters: `skillManager` reloads before elements, then elements before stats; mitigation/on-hit modules reload after scripts. Preserve this ordering unless you understand the dependency chain.
- Player state is centered around `api.player.MMOPlayerData`; startup loads online-player data for `/reload`, schedules periodic online/playing ticks, and flushes offline player data hourly.

## Module pattern

Many managers/features extend `io.lumine.mythic.lib.module.Module` and are annotated with `@ModuleInfo(key = ...)`. The module lifecycle is:

1. `onStartup()` once, before the first enable/reload.
2. `shouldEnable()` to decide whether config enables the module.
3. `onEnable()` when transitioning to enabled.
4. `onReset()` before reloading an already-enabled module or disabling it.
5. `onReload()` after enable/reset when the module remains enabled.
6. `onDisable()` when transitioning to disabled.

Do not make a `Module` itself a Bukkit `Listener`; `Module` explicitly rejects that. Use fields annotated with `@ModuleListener` for listener registration/toggling.

## Version compatibility pattern

Shared code should call `VersionWrapper.get()` or `ServerVersion.get()` instead of directly using version-sensitive Bukkit/NMS APIs. Add or update methods in `mythiclib-plugin/src/main/java/io/lumine/mythic/lib/version/wrapper/VersionWrapper.java` when behavior must vary by server version, then implement the method in each relevant `mythiclib-v...` module.

`ServerVersion.isAbove(...)` and `isUnder(...)` are the normal helpers for feature gates. Watch for post-1.20.5 revision handling and post-1.21.2 API changes noted in wrapper defaults.

## Resources and configuration

- Main plugin metadata is in `mythiclib-plugin/src/main/resources/plugin.yml`; it declares the `mythiclib`/`ml` command, soft dependencies, and load ordering before MMOProfiles/MMOCore.
- Default runtime config is `mythiclib-plugin/src/main/resources/config.yml`.
- Default data files under `mythiclib-plugin/src/main/resources/default/` drive stats, elements, indicators, mitigation types, on-hit effects, triggers, and example scripts/skills.

## Tests

The existing test coverage is in `mythiclib-plugin/src/test/java`, currently focused on `AdventureParserTest`. Tests use JUnit 5 APIs declared in `mythiclib-plugin/pom.xml`.