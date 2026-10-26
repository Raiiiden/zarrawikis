# Version differences

[Back to Zarith](../zarith.md)

`core` is the same on every version. The `common` code is a separate copy per Minecraft version, and the copies differ only where Minecraft changed. Behaviour is meant to be identical everywhere. This page lists the differences that matter when you use the API, write a group subclass, or carry a change across all five copies.

## Families

The copies fall into two families that are nearly identical inside:

| Family | Versions | Mappings | Java |
|---|---|---|---|
| Old | 1.20.1, 1.21.1 | Obfuscated Minecraft, Mojang names with Parchment parameters in dev. Jars are remapped (Fabric) or reobfuscated (Forge). | 17, 21 |
| New | 26.1.2, 26.2, 26.3 | Minecraft ships unobfuscated. Jars are used as built. | 25 |

A change usually lands in 26.3 first and is copied to 26.2 and 26.1.2 almost line for line, then adapted once for 1.21.1 and copied to 1.20.1.

## Loaders

| Version | Loaders | Notes |
|---|---|---|
| 1.20.1 | Forge 47, Fabric | Built with ModDevGradle's legacy Forge plugin. Forge runs on SRG names, so this jar alone carries a mixin refmap, and it bundles MixinExtras through jar-in-jar because Forge 1.20.1 has none. Event listeners name their event class explicitly, because Forge's bus cannot always recover it from a lambda. |
| 1.21.1 and up | NeoForge, Fabric | NeoForge and Fabric Loader ship MixinExtras themselves. |

## API differences you will meet

### Names

| Concept | 1.20.1, 1.21.1 | 26.x |
|---|---|---|
| Resource ids | `ResourceLocation` | `Identifier` |
| Packing an x, z column key | `ChunkPos.asLong(x, z)` | `ChunkPos.pack(x, z)` |
| Lowest build height | `level.getMinBuildHeight()` | `level.getMinY()` |
| Client side check | `level.isClientSide` (field) | `level.isClientSide()` |
| Nearest direction | `Direction.getNearest` | `Direction.getApproximateNearest` |
| Damage | `hurt(source, amount)` | `hurtServer(level, source, amount)` |
| Registry holder lookup | `registryOrThrow(...).getHolderOrThrow(...)` | `lookupOrThrow(...).getOrThrow(...)` |
| Rope default block | `minecraft:chain` | `minecraft:iron_chain` |

### Group subclasses

| | 1.20.1 | 1.21.1 | 26.x |
|---|---|---|---|
| Synced data | `defineSynchedData()` with `entityData.define` | `defineSynchedData(SynchedEntityData.Builder)` | Same as 1.21.1 |
| Saving | Override `readAdditionalSaveData(CompoundTag)` / `addAdditionalSaveData(CompoundTag)`, call `super` | Same as 1.20.1 | Override `readGroup(CompoundTag)` / `writeGroup(CompoundTag)`. Everything sits under `saveKey()` (`"zarith"`). |
| NBT getters | Plain getters (`getInt`, `getIntArray`) with `contains` checks | Same | Optional or defaulted getters (`getIntOr`, `getIntArray(...).orElse(...)`, `getFloatOr`) |
| Right click hook | `interactAt(player, location, hand)` | Same | `interact(player, hand, location)` |
| Block reach | 5 in creative, 4.5 otherwise | `player.blockInteractionRange()` | Same as 1.21.1 |

### Vehicle input before the first packet

Until the driver's client has sent its keys, the server reads vanilla's movement input. On 1.20.1 and 1.21.1 that is the player's `zza`, `xxa` and the `jumping` flag (read through `LivingEntityAccessor`, since the server keeps the jump key there before 1.21.2). On 26.x it is `player.getLastClientInput()`.

### Key bindings

The R and V bindings are created with a translation string category (`key.categories.movement`) on the old family and with `KeyMapping.Category.MOVEMENT` on the new one. Each loader registers them its own way.

## Mixins

| Mixin | 1.20.1, 1.21.1 | 26.1.2 | 26.2, 26.3 |
|---|---|---|---|
| `AbstractArrowMixin`, `EntityCollisionMixin`, `ExplosionMixin`, `FireBlockAccessor`, `LevelMixin`, `PlayerMixin`, `ProjectileUtilMixin`, `ServerGamePacketListenerMixin`, `ServerLevelMixin` | yes | yes | yes |
| `AxeItemAccessor` (axe stripping table) | yes | | |
| `LivingEntityAccessor` (jump flag) | yes | | |
| `ProjectileMixin` (block hits in air blocks) | | | 26.3 only |
| Client: `EntityRendererMixin`, `MinecraftMixin`, `ShadeBrightnessMixin` | yes | yes | yes |
| Client: `GameRendererMixin`, `LevelRendererMixin` (weather drawn through the level renderer) | yes | | |
| Client: `LightCoordsMixin`, `WeatherMixin` | | yes | yes |
| Client: `ClientLevelMixin` (rain splashes under solids) | | | yes |

Where weather and light rendering moved between versions, the mixin moved with it. The behaviour is the same.

Axe stripping on the old family reads vanilla's stripping table through `AxeItemAccessor`. On 26.x stripping became data driven against blocks in the world, so Zarith finds the stripped block by looking up the `stripped_` id of the block instead (logs, wood, stems, hyphae and the bamboo block).

## Rendering

The new family uses Minecraft's render state split: the renderer extracts a `BlockGroupRenderState` each frame and submits from it. The old family draws straight from the entity. `GroupMesh` follows each version's chunk and quad classes. Neither difference shows in the public API, since you only ever register `BlockGroupRenderer::new`.

## Carrying a change across versions

1. Make the change in `core` if it has no Minecraft types in it. It then exists once.
2. Otherwise make it in `versions/26.3/common`, then copy it to 26.2 and 26.1.2. Diff the files afterwards; the copies are expected to match apart from the few renames above.
3. Port it to `versions/1.21.1/common` using the renames on this page, then copy that to 1.20.1.
4. Compile every common project (the command is on [Getting started](getting-started.md#tasks)).

For edits that touch the same spots in all five copies, a small script that asserts each replacement matches exactly once per file is safer than editing by hand.
