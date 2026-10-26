# Zarith

Zarith is the physics library behind my mods. It runs the [Jolt Physics](https://github.com/jrouwe/JoltPhysics) engine inside Minecraft through Stephen Gold's [jolt-jni](https://github.com/stephengold/jolt-jni) bindings, and builds a set of Minecraft features on top of it: blocks that move as rigid bodies, whole builds that fall and break apart, ropes and hinges, and drivable vehicles.

Forge and NeoForge refuse to start when two mods export the same Java packages. Zarith now owns the only copy of jolt-jni in a game and loads its native library once for every mod that needs it.

I wrote this wiki for developers who want to use Zarith from their own mod, or work on Zarith itself. Players only need the [installation section](zarith/getting-started.md#installing-for-players).

## What Zarith gives you

| Area | What it is | Page |
|---|---|---|
| Core physics | One Jolt world per dimension, dynamic bodies built from boxes, the world's blocks mirrored in as static colliders. No Minecraft classes in it. | [Physics core](zarith/physics-core.md) |
| Block groups | An entity holding any number of blocks that move as one body. It collides, floats, burns, can be mined block by block, and splits apart when it loses the blocks holding it together. | [Block groups](zarith/block-groups.md) |
| World interaction | How groups deal with entities, water and lava, fire, tools, explosions and arrows. | [World interaction](zarith/world-interaction.md) |
| Destruction | Structural load, collapse, blast wear, impact damage and debris cleanup. | [Destruction](zarith/destruction.md) |
| Solids API | An interface any moving object can implement so players walk on it, projectiles hit it, and rain stops on it, without writing mixins. | [Solids API](zarith/solids-api.md) |
| Joints | Welds, ball joints, hinges with motors, ropes and springs between bodies or to the world. | [Joints](zarith/joints.md) |
| Vehicles | Cars and motorcycles on Jolt's wheel simulation, plus boats, helicopters, planes, rockets and hover ships. | [Vehicles](zarith/vehicles.md) |
| Sandbox commands | `/zarith` for spawning, assembling, linking, dragging and testing. | [Commands](zarith/commands.md) |

## Supported versions

| Minecraft | Loaders | Java | Notes |
|---|---|---|---|
| 1.20.1 | Forge, Fabric | 17 | Forge jar bundles MixinExtras through jar-in-jar. |
| 1.21.1 | NeoForge, Fabric | 21 | |
| 26.1.2 | NeoForge, Fabric | 25 | Minecraft ships unobfuscated from here on. |
| 26.2 | NeoForge, Fabric | 25 | |
| 26.3 | NeoForge, Fabric | 25 | |

The current version is `0.2.3`. All five Minecraft versions share the same core code and almost the same Minecraft code; the differences are listed in [Version differences](zarith/version-differences.md).

## Pages

Start here:

* [Getting started](zarith/getting-started.md): installing, depending on Zarith, building from source.
* [Architecture](zarith/architecture.md): how the pieces fit, what runs on which thread, the order of one tick.

Reference:

* [Physics core](zarith/physics-core.md): `PhysicsSpace`, `PhysicsBody`, `BodyShape`, `BodySettings`, terrain.
* [Block groups](zarith/block-groups.md): `BlockGroupEntity`, `BlockGrid`, `CellKind`, `PhysicsBlockEntity`, hooks.
* [World interaction](zarith/world-interaction.md): entities, fluids, fire, mining, explosions, arrows.
* [Destruction](zarith/destruction.md): `StructureSolver`, collapse, wear, impacts, debris.
* [Solids API](zarith/solids-api.md): `ZarithSolid` and its helpers.
* [Joints](zarith/joints.md): `JointSettings`, `PhysicsJoint`, joints saved on groups.
* [Vehicles](zarith/vehicles.md): `GroupVehicle`, controls, the dashboard.
  * [Cars and motorcycles](zarith/wheeled-vehicles.md)
  * [Boats](zarith/boats.md)
  * [Helicopters](zarith/helicopters.md)
  * [Planes](zarith/planes.md)
  * [Rockets](zarith/rockets.md)
  * [Hover ships](zarith/hover-ships.md)
* [Commands](zarith/commands.md): the `/zarith` sandbox.
* [Networking and client](zarith/networking-and-client.md): packets, interpolation, rendering, picking.
* [Saved data](zarith/saved-data.md): NBT layout and the settings array format.
* [Version differences](zarith/version-differences.md): what changes between Minecraft versions.
* [Limits and troubleshooting](zarith/limits-and-troubleshooting.md): budgets, performance, common failures.

## Conventions used on every page

**Units.** One block is one metre. Speeds are metres per second, masses kilograms, forces newtons, torques newton metres, angles radians. Densities are kilograms per cubic metre, so a full block with density 500 weighs 500 kg. Vanilla entity motion is in blocks per tick, which is 1/20 of a metre per second value.

**Time.** The physics steps once per server tick, so 20 times a second. `PhysicsSpace.STEP_SECONDS` is `1/20`. Jolt splits each step into 2 collision sub steps.

**Frames.** A block group has its own grid. Block `(x, y, z)` in the grid fills the unit cube from `(x, y, z)` to `(x + 1, y + 1, z + 1)`. The group's body uses that grid as its local frame, so the body's position in the world is where the grid's `(0, 0, 0)` corner is, and its rotation turns the grid. Every point you give a vehicle (wheels, seat, propeller) is in this grid frame.

**Vehicle axes.** In a vehicle's frame +Z is forward and +Y is up. That makes +X point to the vehicle's left, not its right. Turning right is a negative spin about +Y.

**Server authority.** Everything is simulated on the logical server. Clients draw what the server sends and only report input the server cannot see, like mining progress or a player walking into a block.

## Credits

Physics by [Jolt Physics](https://github.com/jrouwe/JoltPhysics) through [jolt-jni](https://github.com/stephengold/jolt-jni) by Stephen Gold. The entity collision code first lived in Felled, my tree felling mod, and I moved it into Zarith so both mods share one set of mixins.
