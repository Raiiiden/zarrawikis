# Networking and client

[Back to Zarith](../zarith.md)

## The channel

All of Zarith's packets travel as raw bytes on one channel, `zarith:main`. `ZarithNetwork` writes and reads the bytes; each loader only carries them across:

| Loader | Transport |
|---|---|
| Forge 1.20.1 | A `SimpleChannel` with protocol version `"1"`. |
| Fabric 1.20.1 | Fabric's play networking on the channel id. |
| NeoForge, and Fabric from 1.21.1 | A `ZarithPayload` record holding the bytes, registered both ways. On NeoForge the registrar version is `"1"`. |

The first byte of every packet is its type. Server and client types are numbered separately.

Loader entrypoints hand `ZarithNetwork` three senders: to one player, to every player tracking an entity, and (on the client) to the server. If you port Zarith to another loader, that and calling `onServer` / `onClient` with the bytes is all the networking you need.

## Client to server

| Type | Name | Contents | Server does |
|---|---|---|---|
| 0 | `DROP` | | Drops what the player holds in drag mode. |
| 1 | `THROW` | | Throws it. |
| 2 | `MINE` | group id, cell id | `mineCell`: checks reach and timing, then breaks the block. |
| 3 | `PUSH` | group id, direction x, z | `push`: the player walked into the group. |
| 4 | `LAND` | group id, speed | `landed`: the player landed on the group. Speed capped at 30 m/s. |
| 5 | `USE` | group id, cell id, off hand | `useOnCell`: axe or lighter on a block. |
| 6 | `PUT_OUT` | group id, fire spot | `putOutFire`. |
| 7 | `VEHICLE_INPUT` | group id, accelerate, right, handbrake, lift, gear shifts | `vehicleInput`, ignored unless the sender is the driver. |

The server ignores any packet naming an entity more than 64 blocks from the sender, and every action rechecks the player's reach itself.

## Server to client

| Type | Name | Contents | Sent |
|---|---|---|---|
| 0 | `DRAG_MODE` | on | When the player's drag mode changes. |
| 1 | `PIN` | arrow id, group id, local position, local heading | When an arrow sticks in a group; to players within 160 blocks, and to a player who starts tracking the group. |
| 2 | `UNPIN` | arrow id, optional motion | When an arrow leaves a group. With motion, it flies off. |
| 3 | `GROUP_SPAWN` | group id, spawn data | To a player who starts tracking the group. |
| 4 | `GROUP_POSE` | group id, game time, origin, rotation, solid | Every tick the group is awake, the tick it falls asleep, and when its solidity changes. |
| 5 | `GROUP_REMOVED` | group id, cell ids | When blocks leave the group. |
| 6 | `GROUP_STATE` | group id, cell id, block state id | When one block changes in place (stripped, scraped). |
| 7 | `GROUP_FIRES` | group id, fire spots | When the set of fire spots changes. |
| 8 | `GROUP_WHEELS` | group id, speed, rpm, gear, three values per wheel | Every tick a vehicle is awake. |

Everything except `DRAG_MODE`, `PIN` and `UNPIN` goes to all players tracking the group.

### Spawn data

Vanilla's entity spawn packet cannot carry a grid of blocks, so `GROUP_SPAWN` follows it. It holds the whole grid (bricks in slot order so ids match on both sides, each brick's palette, its occupied layers with all four masks, and a palette index per block when there is more than one state), the origin and rotation, whether the group is solid and whether its soft blocks are, the fire spots, and the vehicle if it has one (seat, wheel block, every settings array). Removed blocks are included, so ids line up with the server's.

### Wheels and dashboard

`GROUP_WHEELS` packs every value into a short: speed to 0.01 m/s, angles to 0.0001 rad, suspension to 0.1 mm, rpm as a whole number, and gear as a byte. Wheel roll angles are wrapped into one turn before sending; the client unwraps them when interpolating, so a wheel never spins backward between ticks.

## Poses on the client

The client never simulates a group. It plays back the poses the server sends.

* Each `GROUP_POSE` is stored as a snapshot stamped with the server's game time, up to 40 of them.
* The group is drawn about 2 ticks behind the newest snapshot, interpolating position linearly and rotation with slerp. Packets that arrive unevenly still give even motion.
* The offset between the client's tick count and the server's clock is smoothed by 5% per packet, and reset if it is ever more than 3 ticks off.
* After a gap of more than 3 ticks (the group slept, or packets stopped), a copy of the last pose is inserted first, so the group holds still and then moves instead of sliding slowly across the gap.
* A pose that moves less than a hair is treated as still. A resting group does not wobble, its collision boxes are not rebuilt, and after 20 still ticks it starts casting shade.

`renderPose(partialTick, origin, rotation)` returns the interpolated pose for a frame. On the server it interpolates between the last two ticks instead.

## Predicting breaks

When the local player breaks a block on a group, the client removes it at once, plays the break sound and particles itself, and tells the server. If the server's `GROUP_REMOVED` for that block has not arrived within 30 ticks, the block comes back. The server leaves the breaking player out of its own break effects, as vanilla does.

## Picking

After vanilla works out what the crosshair is on, `ZarithClient.pick` casts a ray through every nearby group:

1. The ray is turned into each group's grid frame at its drawn pose.
2. It steps through the grid cell by cell (up to 256 cells, within the player's block reach).
3. A live block is hit only on its outline shape, so you can aim past a slab or a fence post.
4. An empty cell holding fire is hit as a whole cell, so fire can be punched out.

If a group block or fire spot is closer than what vanilla found, it becomes the target: `mc.hitResult` is set to an entity hit on the group, and `ZarithClient.target()` holds the group, cell id, hit point and fire spot. The renderer draws the outline and crack stage on the target block.

Groups the local player is holding are skipped.

## Rendering

`BlockGroupRenderer` draws any `BlockGroupEntity` subclass. Register it for your own entity type with `BlockGroupRenderer::new`.

The mesh (`GroupMesh`) is built from each block's baked model quads:

* A face is culled when the group has an opaque full block right against it. Faces between two soft blocks are culled when `cullsInnerSoft()` returns `true`.
* Biome tint is taken from where the group is.
* Smooth light and ambient occlusion are sampled from the world around the group's current pose, so a group moving into a cave darkens.
* Quads are packed per 8 by 8 by 8 brick and per facing. Each frame only bricks inside the view frustum are drawn, and within them only faces that can face the camera.
* Lighting is refreshed in slices: a moving group relights all its bricks (a big one every few ticks), changed bricks relight at once, and a still group relights a slice of its bricks a tick and all of them every two seconds.
* Block entities with special renderers (chests, beds, signs) have no baked quads, so they are drawn as models one by one, the way minecarts draw the block they carry.

On top of the mesh it draws:

* a vanilla fire block in each fire spot, turned to the grid axes nearest the world's and with the side flags fire would have there,
* a vehicle's wheels, as two copies of `wheelState` a quarter turn apart so they read as a rough disc, rolling, steering and riding their suspension between the last two server ticks,
* the target block's outline and crack.

Groups cast no vanilla entity shadow. They are drawn as far away as a group 64 times their size would be, at least 4 blocks of size, scaled by the entity view distance setting.

## Client tick hooks

| Hook | Runs | Does |
|---|---|---|
| `ZarithClient.tickStart` | Start of `Minecraft.tick` | Every solid advances its pose (`SolidMotion.advanceAll`). |
| `PlayerMixin` | Start of the local player's tick | Carry and push out (`SolidMotion.beforeLocalPlayerTick`). |
| `ZarithClient.tickEnd` | End of `Minecraft.tick` | Mining, `VehicleControls`, `SolidEffects` (fire smoke, rain splashes), `SolidShade`. |
| `ZarithClient.startAttack` | Left click | Throws a held group, punches out fire, or starts mining a group block. Returns `true` to cancel vanilla's attack. |
| `ZarithClient.startUseItem` | Right click | Drops a held group, or uses an axe or lighter on a group block. Returns `true` to cancel vanilla's use. |
| `ZarithClient.pick` | After vanilla's pick, each frame | Places stuck arrows, finds the target. |

The tick hooks do nothing while the game is paused.
