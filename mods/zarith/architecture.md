# Architecture

[Back to Zarith](../zarith.md)

## Layers

Zarith is three layers stacked on each other. Each one only knows about the layers below it.

```text
loader entrypoint      ZarithForge / ZarithNeoForge / ZarithFabric (+ their client classes)
      |                registers the entity, the network channel, commands and the tick hook
      v
common (per version)   BlockGroupEntity, GroupVehicle, LevelPhysics, the solids API, mixins,
      |                commands, networking, renderer. Uses Minecraft, never a loader.
      v
core (shared)          ZarithJolt, PhysicsSpace, PhysicsBody, joints, vehicle controllers,
                       StructureSolver. Uses jolt-jni, JOML and fastutil, never Minecraft.
```

`core` sees the world only through the `TerrainSource` interface, which each version's `common` implements as `LevelTerrain`. That is the whole bridge in that direction. In the other direction, `common` holds a `PhysicsBody` per entity and sets the entity as the body's owner with `PhysicsBody.setOwner`.

Because `core` has no Minecraft in it, its classes are identical on every Minecraft version and are compiled into every jar. Anything that can be written without Minecraft types should go there.

## Main classes and who owns what

| Class | Layer | Lifetime | Owns |
|---|---|---|---|
| `ZarithJolt` | core | Whole JVM | The loaded native, Jolt's global factory, one shared job system and one shared 32 MB temp allocator. |
| `PhysicsSpace` | core | One per server level, made on demand | A Jolt `PhysicsSystem`, every body in it, the terrain colliders, joints and wheeled vehicles. |
| `PhysicsBody` | core | Until destroyed | One Jolt body. Knows its space, its mass, its joints and its wheeled vehicle if it has one. |
| `LevelPhysics` | common | Static | The map from `ServerLevel` to `PhysicsSpace`. |
| `BlockGroupEntity` | common | Vanilla entity lifetime | A `BlockGrid`, its `PhysicsBody` (server only), its `GroupJoints`, its `GroupVehicle`. |
| `GroupVehicle` | common | Lives on its group | Settings for every vehicle type, and the controllers that act on the group's body. |
| `SolidIndex` | common | One per `Level`, stored on the level by a mixin | Every registered `ZarithSolid` in that level, with a column grid for fast lookup. |

### Physics spaces

`LevelPhysics.get(level)` returns the level's space and makes it the first time. It returns `null` only when Jolt cannot load on this machine. `LevelPhysics.getIfPresent(level)` never creates one, and returns `null` on the client.

A space is named after its dimension (`minecraft:overworld` and so on), which shows up in the log. It is destroyed when its level unloads and all spaces are destroyed when the server stops. Destroying a space frees every Jolt object in it; any `PhysicsBody` you still hold reports `isDestroyed()` and every method on it becomes a no-op.

### Bodies and entities

A block group creates its body lazily on its first server tick, through `LevelPhysics.get` and `PhysicsSpace.createBody`. The body is never saved. What is saved is the group's pose, its velocity and spin, and whether it was asleep, and a new body is built from those after loading.

Bodies are removed in two places:

* When the entity leaves its level for any reason (discarded, removed by a command, unloaded with its chunk), `BlockGroupEntity` hears about it through the level's own entity callback and destroys its body.
* At the start of `LevelPhysics.tick`, any body whose owner is a removed `BlockGroupEntity`, or one now in a different level, is destroyed. This catches anything the first path missed.

If you hold a body for your own entity type, call `PhysicsBody.setOwner(this)` and destroy the body yourself when the entity goes. `LevelPhysics` only cleans up after `BlockGroupEntity` owners.

## One server tick

Minecraft ticks a level's entities, then the loader's end of level tick hook calls `LevelPhysics.tick(level)`, and that steps the space. So every group reads the result of the previous step at the start of its tick, decides what to do, and its forces act in the step at the end of the same tick.

**1. Each `BlockGroupEntity` ticks** (`serverTick`), in this order:

1. Discards itself if it fell more than 64 blocks below the world, or has no blocks left.
2. If it has no solid blocks left, calls `noSolidsLeft` (by default it breaks into drops) and stops.
3. Makes its body if it has none.
4. Builds any joints whose other end has appeared (`GroupJoints.tick`).
5. Runs its vehicle (`GroupVehicle.tick`): reads the driver's input and has each controller add its forces and spin for the coming step.
6. Finds entities standing on it, then reads its pose, velocity and spin from the body.
7. Updates its bounds and collision boxes, carries the riders by however far the point under their feet moved, and pushes entities out of its way.
8. Turns any vanilla knockback it received into body velocity.
9. Calls `onPose`, lets mobs walking into it push it.
10. Puts the body to sleep if it has crept along slowly for 30 ticks (see [Sleeping](physics-core.md#sleeping)).
11. While awake: detects impacts, crushes entities, calls `onAwake`.
12. Counts down debris lifetime, applies impact breaks and structural collapse.
13. Calls `onSettle`, applies water and lava forces, ticks fire, carries stuck arrows.
14. Sends its pose to watching clients if it is awake or just fell asleep, and its wheels and dashboard if it is a vehicle.
15. Calls `onEndTick`.

**2. `LevelPhysics.tick(level)` runs** after all entities:

1. Destroys bodies whose owners are gone (above).
2. Steers bodies held in drag mode (`Grabs.tick`).
3. Calls `PhysicsSpace.step()`.

**3. `PhysicsSpace.step()`:**

1. Every 20 ticks, drops terrain colliders nobody has used for 100 ticks.
2. Returns early if the space is paused or empty.
3. For each body, updates the block region it needs mirrored, and on schedule re-checks those blocks for changes.
4. Rebuilds terrain sections that changed, refreshes joint collision filters, and optimizes Jolt's broad phase after 512 new static bodies.
5. Hands each wheeled vehicle its eased throttle, steering and brakes.
6. Steps Jolt by `1/20` s in 2 collision sub steps on the shared job system.

Forces added with `PhysicsBody.addForceAndTorque` last for exactly one step. They are also ignored while the body is asleep, so every controller that should wake a sleeping body calls `activate()` first.

## One client tick

The client never simulates. It plays back poses the server sends and handles the local player against them.

1. **Start of `Minecraft.tick`** (`ZarithClient.tickStart`): every registered solid advances to this tick's pose with `clientAdvance`. If the local player stands on one, the distance that carried the player is remembered.
2. **Start of the local player's tick** (`SolidMotion.beforeLocalPlayerTick`): the player is moved by that carry, against terrain, then every nearby solid pushes the player out of itself if it moved into them. Only after this does the player's own movement run, and that movement collides with solids through the entity collision mixin.
3. **End of `Minecraft.tick`** (`ZarithClient.tickEnd`): mining progress on group blocks, vehicle key input, fire and rain particles, and the sky light shade map.
4. **Each frame, after vanilla's pick** (`ZarithClient.pick`): arrows stuck in groups are placed on the group's drawn pose, then a ray through every nearby group finds the block or fire spot under the crosshair.

Players are the one exception to server authority over movement. Vanilla lets clients move their own player, and Zarith keeps it that way: collision with solids runs on the client for the local player and is skipped for players on the server, which would otherwise disagree with the client and rubber band. Every other entity collides with solids on the server.

## Threads

| Work | Thread |
|---|---|
| Everything in `PhysicsSpace` and `PhysicsBody` | Server thread only. Spaces are stepped one at a time. |
| Jolt's own solver | A shared `JobSystemThreadPool` with `min(4, cores / 2)` worker threads, at least one. |
| Structural load checks | One daemon thread named `Zarith structure`. The solve works on a copy and the result is applied on the server thread. |
| Sky light shade map | Built on the client thread and read by chunk meshing threads. It is swapped as a whole immutable snapshot, never edited in place. |

Do not call into a `PhysicsSpace` or `PhysicsBody` from another thread. Jolt's body interface is used without locks (`getBodyInterfaceNoLock`) because only the server thread touches it.

## Why the solids API exists

Felled had mixins of its own for collision with fallen trees, and Zarith needed the same mixins for physics blocks. Two mods redirecting the same vanilla call crash the game, so I moved the mixins into Zarith and turned them into an interface: anything that implements `ZarithSolid` and registers itself gets entity collision, projectile hits, explosion shielding, rain cover and sky light shading from Zarith's one set of mixins. See [Solids API](solids-api.md).
