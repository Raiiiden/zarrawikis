# Getting started

[Back to Zarith](../zarith.md)

## Installing for players

Put the Zarith jar for your Minecraft version and loader in the `mods` folder. The jar names follow `zarith-<loader>-<minecraft>-<version>.jar`, for example `zarith-neoforge-26.3-0.2.3.jar` or `zarith-forge-1.20.1-0.2.3.jar`. On Fabric you also need Fabric API.

The jar carries the Jolt native library for these platforms:

| OS | Architectures |
|---|---|
| Windows | x86-64 |
| Linux | x86-64, ARM64 |
| macOS | x86-64, Apple Silicon (ARM64) |

On first use Zarith copies the right native into `<game directory>/.zarith/natives/6.0.0/`. It uses the game directory instead of the system temp folder because antivirus scanners and temp cleaners tend to lock or delete files there. The folder can be deleted at any time; it is recreated on the next launch.

On a platform with no native (32-bit Windows, for example) the game still starts. Physics is simply unavailable, `/zarith stats` and the commands that need physics report why, and mods that depend on Zarith get `null` back where they would have received a physics space.

Zarith must be installed on both the server and every client. The server runs the simulation; clients need Zarith to draw block groups and to send the input the server cannot see.

## Depending on Zarith from your mod

### The one rule

**Never bundle jolt-jni in your own jar, and never put it on your classpath as a separate library.** Zarith's jar already contains jolt-jni's classes. If a second jar exports the `com.github.stephengold.joltjni` packages, Forge and NeoForge refuse to launch with a split package error. On Fabric the game may start, but the jolt-jni classes then come from a different class loader than Zarith's. Zarith checks for exactly this: `ZarithJolt.load()` fails with a message saying another mod probably bundles its own jolt-jni, and physics stays off.

The reason this is strict: JNI attaches a native library to the class loader of whichever class called `System.load`. Zarith calls it from `ZarithJolt`, which lives in the same jar as the jolt-jni classes, so they share a class loader. That only works if Zarith's copy is the only copy.

### Getting the artifacts

Zarith is published to your local Maven repository by its own build (see [Building from source](#building-from-source)). The coordinates are:

| Artifact | What it holds | Use it for |
|---|---|---|
| `com.raiiiden.zarith:zarith-<loader>-<mc>:<version>` | The full mod jar for one loader. Remapped or reobfuscated where that Minecraft version needs it. | Running the game in your dev environment. |
| `com.raiiiden.zarith:zarith-common-<mc>:<version>` | Zarith's core and that version's vanilla-only classes, in Mojang names, never reobfuscated. | Compiling a vanilla-only `common` project of your own. |

`<loader>` is `forge`, `neoforge` or `fabric`. `<mc>` is `1.20.1`, `1.21.1`, `26.1.2`, `26.2` or `26.3`.

The published POMs list no dependencies on purpose. A consumer never pulls jolt-jni in as a library; it gets jolt-jni's classes through the Zarith jar.

A Gradle setup looks like this. Adjust the configuration name to your build plugin.

```groovy
repositories {
    mavenLocal {
        content { includeGroup 'com.raiiiden.zarith' }
    }
}

dependencies {
    // ForgeGradle on 1.20.1
    implementation fg.deobf("com.raiiiden.zarith:zarith-forge-1.20.1:0.2.3")

    // ModDevGradle on NeoForge
    implementation "com.raiiiden.zarith:zarith-neoforge-26.3:0.2.3"

    // A vanilla-only common project
    compileOnly "com.raiiiden.zarith:zarith-common-26.3:0.2.3"
}
```

### Declaring the dependency

Declare `zarith` as a required dependency that loads before your mod.

`mods.toml` (Forge 1.20.1):

```toml
[[dependencies.yourmod]]
    modId="zarith"
    mandatory=true
    versionRange="[0.2.3,)"
    ordering="AFTER"
    side="BOTH"
```

`neoforge.mods.toml`:

```toml
[[dependencies.yourmod]]
    modId="zarith"
    type="required"
    versionRange="[0.2.3,)"
    ordering="AFTER"
    side="BOTH"
```

`fabric.mod.json`:

```json
"depends": {
  "zarith": ">=0.2.3"
}
```

### Loading Jolt

If you only use block groups, vehicles or the solids API, you never touch Jolt yourself and there is nothing to load. `LevelPhysics.get(level)` loads Jolt the first time any level needs physics.

If you call jolt-jni classes directly, call this first:

```java
if (!ZarithJolt.load()) {
    LOGGER.warn("No physics: {}", ZarithJolt.failureReason());
    return;
}
```

`ZarithJolt.load()` is safe to call from any mod, from any thread, any number of times. The first call does the work and later calls return the same answer. It returns `false` with a reason instead of throwing, so a missing native turns into "no physics" rather than a crash. Calling any jolt-jni method before `load()` has returned `true` crashes the JVM, because the native methods are not bound yet.

## A first block group

This puts a 3 by 1 by 3 slab of stone in the air above a player. It falls, lands, and can then be mined, dragged, blown up and set on fire like any physics block.

```java
ServerLevel level = (ServerLevel) player.level();
BlockGrid grid = new BlockGrid();
for (int x = 0; x < 3; x++) {
    for (int z = 0; z < 3; z++) {
        grid.add(x, 0, z, Blocks.STONE.defaultBlockState(), CellKind.SOLID);
    }
}
Vector3d corner = new Vector3d(player.getX() - 1.5, player.getY() + 5, player.getZ() - 1.5);
PhysicsBlockEntity slab = PhysicsBlockEntity.create(level, grid, corner, new Quaternionf(), PhysicsBlockEntity.COLLISION);
level.addFreshEntity(slab);
```

`corner` is where the grid's `(0, 0, 0)` corner goes in the world. `PhysicsBlockEntity.COLLISION` makes entities stand on it; leave it out and players walk through it while it still collides with the terrain and other bodies. The [Block groups](block-groups.md) page covers subclassing `BlockGroupEntity` when you want your own entity type with its own behaviour.

## Building from source

### Requirements

The build runs Gradle 9.8 on a Java 25 daemon. `gradle/gradle-daemon-jvm.properties` asks for JDK 25 and Gradle finds a local one by itself. Each subproject still compiles with its own toolchain (17, 21 or 25), so the daemon version does not change what the jars target. Fabric Loom 1.18 is the reason the daemon has to be 25.

### Project layout

```text
core/                     no Minecraft, no loader; compiled into every jar (Java 17)
  jolt/ZarithJolt         loads the native, owns the shared job system and allocator
  physics/                PhysicsSpace, PhysicsBody, joints, every vehicle controller
  structure/              StructureSolver
versions/<mc>/
  gradle.properties       that version's Minecraft, loader and Java versions
  common/                 vanilla-only code: entities, renderer, commands, mixins, solids API
  forge | neoforge/       loader entrypoint and mods.toml
  fabric/                 loader entrypoint and fabric.mod.json
gradle/
  zarith-java.gradle      toolchains and repositories
  zarith-common.gradle    applied by each common project; publishes zarith-common-<mc>
  zarith-loader.gradle    turns a loader project into a Zarith jar
```

`common` projects are vanilla only on purpose. They compile against Minecraft with no loader classes on the classpath, so nothing loader-specific can leak into shared code. Loader entrypoints are thin: they register the entity type, wire the network channel, register commands, and call `LevelPhysics.tick` at the end of each level tick.

### How a jar is assembled

A loader jar is not built from dependencies. `zarith-loader.gradle` compiles three source trees together in one `compileJava`: the loader project, its version's `common`, and `core`. It then unpacks jolt-jni's classes straight into the class output and adds every platform's native library to the jar.

Compiling the sources together, instead of depending on `core` and `common` as libraries, is what keeps `ZarithJolt` and the jolt-jni classes on one class loader in a ModDevGradle dev run. A separate library jar would land on a different class loader and `System.load` would bind the natives to the wrong one.

`module-info.class` from jolt-jni is excluded when unpacking. jolt-jni is a named module, and leaving its descriptor in the class output would turn the whole output into that module and hide Minecraft from the compiler on the next build.

### Tasks

```text
gradlew buildAll                        every loader jar, copied to build/dist
gradlew publishAllToMavenLocal          every loader jar and common jar to ~/.m2
gradlew :26.3:neoforge:build            one target
gradlew -Pzarith.only=26.3 build        configure only one Minecraft version
gradlew -Pzarith.only=1.20.1,26.3 build configure two
gradlew :1.20.1:forge:runClient         dev client; runServer works the same way
```

`-Pzarith.only` matters more than it looks. Every Loom and ModDevGradle project sets up its Minecraft while Gradle configures, so a plain build configures ten loader projects and five common ones before it compiles anything.

To only type check after changing code, compile the common projects:

```text
gradlew -Pzarith.only=1.20.1,1.21.1,26.1.2,26.2,26.3 :core:compileJava :1.20.1:common:compileJava :1.21.1:common:compileJava :26.1.2:common:compileJava :26.2:common:compileJava :26.3:common:compileJava --offline -q
```

### Changing shared code

The five `versions/<mc>/common` folders are separate copies, not one source set with conditionals. A change to `GroupVehicle`, `BlockGroupEntity` or anything else under `common` has to be made five times. Most files are identical between 26.1.2, 26.2 and 26.3, and between 1.20.1 and 1.21.1, apart from Minecraft API renames. [Version differences](version-differences.md) lists the renames you will meet.

Code that has no Minecraft in it belongs in `core`, which exists once.
