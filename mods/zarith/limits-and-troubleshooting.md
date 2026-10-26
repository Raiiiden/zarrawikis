# Limits and troubleshooting

[Back to Zarith](../zarith.md)

## Hard limits

| Limit | Value | Where |
|---|---|---|
| Bodies per dimension | 65,536, terrain section bodies included | `PhysicsSpace` |
| Body pairs | 65,536 | `PhysicsSpace` |
| Contact constraints | 32,768 | `PhysicsSpace` |
| Speed of any body | 120 m/s | `PhysicsSpace` |
| Terrain mirrored around one body | 96 blocks per axis | `PhysicsSpace` |
| Blocks in `/zarith physicify` | 4,096 (box up to 16,384) | `ZarithCommands` |
| Blocks in `/zarith assemble` | 32,768 (box up to 131,072) | `Contraptions` |
| Rope length | 0.5 to 64 blocks | `Contraptions` |
| Cloth | 16 by 16 tiles | `ZarithCommands` |
| Structural check | Groups up to 60,000 solid blocks | `BlockGroupEntity.maxStructureBlocks` |
| Packet range | Entities within 64 blocks of the sender | `ZarithNetwork` |
| Settings arrays in packets | 256 values each | `GroupVehicle.readSpawn` |
| Wheels sent per vehicle | 256 values (85 wheels) | `ZarithNetwork` |

The `/zarith` limits are only on the commands. From code you can build bigger groups, within the body and memory limits.

## Performance

Things that cost the most, roughly in order:

* **Awake bodies.** A sleeping body costs almost nothing. Most slowdowns come from many bodies that never settle. Check `/zarith stats`: it shows how many bodies are awake and how long the last step took.
* **Many single block groups.** `/zarith physicify` on a big box makes one group per block, each with its own entity, body, terrain region and packets. `/zarith assemble` makes one group of the same blocks and is far cheaper until it breaks apart. Debris cleanup exists so demolitions do not leave thousands of tiny groups behind.
* **Terrain churn.** A body's terrain region follows it. Fast bodies over complex terrain (lots of non cube blocks, stairs, slabs, fences) rebuild more sections. Flat ground and solid stone are cheap, because full cubes merge and buried ones are skipped.
* **Joints.** Long ropes and big cloths are many small bodies held by many constraints, and Jolt has to solve them all every step.
* **Client meshes.** Big groups are drawn brick by brick and only visible faces are drawn, but a moving group relights every brick, and a large one every few ticks.

Groups being still is what makes everything cheap: they sleep, stop sending poses, stop relighting, and start casting shade. Anything that keeps a group slightly awake (a controller calling `activate()` every tick, a force that never balances) costs a full step for that body every tick.

## Problems

### "Jolt physics is unavailable"

`/zarith stats` and the log say why. `ZarithJolt` logs `Zarith: Jolt physics unavailable: <reason>`.

| Reason starts with | Cause | Fix |
|---|---|---|
| `no Jolt native is bundled for os.name=...` | An OS or CPU with no native, such as 32 bit Windows or a RISC-V board. | None; physics is not available there. |
| `jolt-jni classes were loaded by ... Another mod probably bundles its own jolt-jni` | Another jar carries jolt-jni. | Find the mod that bundles `com.github.stephengold.joltjni` and remove it, or have it depend on Zarith instead. |
| `UnsatisfiedLinkError` | The native could not be loaded: a damaged file, or missing system libraries on Linux. | Delete `<game dir>/.zarith/natives/` and launch again. On Linux, check the C++ runtime is installed. |
| `bundled native not found on the classpath` | A Zarith jar built without its natives. | Use a release jar from `buildAll`, not a `common` jar. |

### The game will not start: split package or module conflict

Forge and NeoForge refuse to launch when two mods export the same packages. If the error names `com.github.stephengold.joltjni`, a mod other than Zarith bundles jolt-jni. Only Zarith may carry it.

### A group floats in the air

It fell asleep there. Sleeping bodies feel no gravity. Usually something removed the support under it in a way the terrain check has not seen yet, and it falls a tick or two later. If it stays:

* Is the space paused? `/zarith stats` says `PAUSED`.
* Is it a parked hover ship? That is intended.
* Was the block under it changed in an unloaded chunk? Terrain in unloaded chunks is not mirrored, and the group wakes when it is checked again.

### A body falls through the world at chunk edges

Unloaded chunks are left out of the terrain, so a body moving into an unloaded chunk has no ground there. Keep physics where players are, or keep the chunks loaded.

### Players rubber band on a moving group

Player collision with groups runs on the client against the pose the client has drawn, which is about two ticks behind the server. The server does not collide players with solids at all, so it agrees with whatever the client did. If you see rubber banding, something on the server is moving the player: check for another mod that moves players or a server side anti cheat plugin.

### My subclass's groups are invisible

The renderer is registered per entity type. Register `BlockGroupRenderer::new` for your own type in your client setup.

### My vehicle does not respond

* The group must have its vehicle set before it is spawned, or clients never learn the seat.
* Only the first passenger drives.
* Planes and rockets only run their engine with a player in the seat. A plane needs airspeed above `controlSpeed` for full control and has none standing still.
* A boat only moves where its propeller point is in water; a helicopter only does anything once airborne.
* `addForceAndTorque` is ignored on a sleeping body. Controllers wake their body when the pilot presses a key, but your own code must call `activate()` first.

### Saved vehicle settings changed after an update

The settings arrays only ever grow at the end. If a value moved, someone reordered an array; see [Saved data](saved-data.md#the-settings-array-format).

### A joint does not come back after reloading

Joints between groups are made again only when both groups are loaded. The holding group retries every 20 ticks. If the other group was destroyed rather than unloaded, the joint is gone for good.

## Reporting a bug

Include the Minecraft version, the loader and its version, the Zarith version, the output of `/zarith stats` in the dimension where it happens, and the log lines starting with `Zarith:`.
