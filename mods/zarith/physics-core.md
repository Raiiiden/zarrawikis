# Physics core

[Back to Zarith](../zarith.md)

Everything on this page lives in `core` (packages `com.raiiiden.zarith.core.jolt` and `com.raiiiden.zarith.core.physics`). None of it imports Minecraft, so it is identical on every version. Most mods never use it directly; block groups and vehicles drive it for you. Use it when you need a body that is not made of blocks, or when you write your own entity around a body.

## ZarithJolt

`com.raiiiden.zarith.core.jolt.ZarithJolt` loads Jolt once per JVM.

| Method | What it does |
|---|---|
| `load()` | Extracts and loads the native, registers Jolt's allocator, factory and types. Returns `true` when ready. Safe from any thread, any number of times; only the first call does work. Never throws. |
| `isReady()` | `true` once `load()` succeeded. |
| `failureReason()` | Why loading failed, or `null`. |
| `versionString()` | Jolt's version, or `"unavailable"`. |
| `setGameDirectory(Path)` | Where `.zarith/natives` goes. Each loader entrypoint calls this through `Zarith.init`. A mod that loads Jolt before Zarith's entrypoint runs still gets the right folder, because `ZarithJolt` falls back to asking FML or Fabric Loader for the game directory by reflection. |
| `sharedJobSystem()`, `sharedTempAllocator()` | The job system and 32 MB temp allocator every space steps with. Throw if Jolt is not loaded. |
| `LOGGER` | The `Zarith` SLF4J logger. |

`load()` fails, with a reason, in these cases: no native for this OS and architecture, the native could not be extracted or loaded, or the jolt-jni classes came from a different class loader than Zarith (another mod bundles jolt-jni).

If the native file from an earlier launch is already in `.zarith/natives/6.0.0/` and has the same size as the bundled one, it is reused. On Windows a DLL that another game instance has loaded cannot be replaced; Zarith treats the file already there as correct in that case.

## PhysicsSpace

One Jolt world. `LevelPhysics` makes one per server level; you can also make your own.

```java
PhysicsSpace space = new PhysicsSpace("my-test-space", terrain); // terrain may be null
```

The constructor throws `IllegalStateException` if Jolt is not loaded. A space with `null` terrain has no world blocks in it, only the bodies you add.

### Constants and budgets

| Name | Value | Meaning |
|---|---|---|
| `STEP_SECONDS` | `1/20` | Length of one step. Public; controllers use it. |
| Collision sub steps | 2 | Jolt collision steps per tick. Keeps small stacked blocks from jittering apart. `subStepSeconds()` returns `1/40`. |
| Max bodies | 65,536 | Including terrain section bodies. `createBody` returns `null` and logs an error past it. |
| Max body pairs | 65,536 | |
| Max contact constraints | 32,768 | |
| Max linear velocity | 120 m/s | Per body. |
| Penetration slop | 0.01 m | |
| Default gravity | (0, -9.81, 0) | Per space. `/zarith gravity` changes it. |

### Collision layers

Each body sits on one of four object layers.

| Layer | Used by | Collides with |
|---|---|---|
| `STATIC` | Terrain | `MOVING`, `PASSING` |
| `STATIC_SOFT` | Soft terrain (leaves, anything in `#zarith:soft_terrain`) | `MOVING` only |
| `MOVING` | Normal bodies | Everything |
| `PASSING` | Bodies with `BodySettings.passesSoftTerrain` | `STATIC`, `MOVING`, `PASSING` |

So a body that passes soft terrain falls through leaves, while everything else lands on them. Felled uses this for falling trees.

### Methods

| Method | Notes |
|---|---|
| `createBody(BodyShape, BodySettings)` | Adds a dynamic body. Returns `null` if no box in the shape collides, if the shape cannot be built, or if the body budget is full. |
| `destroyBody(PhysicsBody)` | Also destroys the body's joints and wheeled vehicle, and releases its terrain. Same as `body.destroy()`. |
| `bodies()` | Unmodifiable list of live bodies. |
| `reshape(PhysicsBody, BodyShape)` | Swaps the shape in place, keeping the body's origin where it is. Rebuilds the body's joints and vehicle so they stay correct about the new centre of mass. Returns `false` and changes nothing if no box in the new shape collides. |
| `createJoint(a, b, JointSettings)` | See [Joints](joints.md). `b` may be `null` to pin `a` to the world. |
| `destroyJoint(PhysicsJoint)`, `joints()` | |
| `createVehicle(PhysicsBody, VehicleSettings)` | Adds Jolt's wheeled vehicle. A body has at most one; creating a second replaces the first. Returns `null` without wheels. See [Cars and motorcycles](wheeled-vehicles.md). |
| `destroyVehicle(PhysicsVehicle)` | |
| `step()` | Advances one tick. `LevelPhysics` calls it; call it yourself only for a space you made. |
| `setPaused(boolean)`, `isPaused()` | A paused space keeps every body where it is. Resuming wakes all bodies. |
| `gravity()`, `setGravity(x, y, z)` | Per space. Setting it wakes all bodies. |
| `explode(x, y, z, radius, strength)` | Adds velocity (not impulse) away from the point to every body whose bounds centre is within `radius`: `strength` m/s at the centre falling to nothing at the edge, plus a quarter of that upward and a random spin. Returns how many bodies it pushed. This is what `/zarith explode` uses. It breaks nothing; vanilla explosions go through a different path (see [World interaction](world-interaction.md#explosions)). |
| `onBlockChanged(x, y, z)` | Rebuilds the terrain colliders at and around a block and wakes sleeping bodies near it. You never need to call this, because changes are found on their own within a tick or two. Call it from a block change hook if you need the reaction on the same tick. |
| `bodyCount()`, `activeBodyCount()`, `jointCount()`, `vehicleCount()`, `terrainCellCount()`, `terrainColliderCount()`, `lastStepMillis()` | Statistics. `/zarith stats` prints them. |
| `destroy()`, `isDestroyed()` | Frees everything. The space is unusable afterwards. |

## BodyShape

A body's shape is a list of axis aligned boxes in the body's local frame. The body's position is this frame's origin.

```java
BodyShape crate = BodyShape.box(0.5f, 0.5f, 0.5f);        // a cube centred on the origin
BodyShape cube = BodyShape.cube();                         // the same

BodyShape table = new BodyShape()
        .add(0, 0.8, 0, 2, 1, 1)                           // top, settings density
        .add(0, 0, 0, 0.1, 0.8, 0.1)                       // one leg
        .add(1.9, 0, 0, 2, 0.8, 0.1)
        .add(0, 0, 0.9, 0.1, 0.8, 1)
        .add(1.9, 0, 0.9, 2, 0.8, 1)
        .add(0, 0, 0, 2, 0.3, 1, 2000f, false);            // heavy, mass only: lowers the centre of mass
```

`add(minX, minY, minZ, maxX, maxY, maxZ)` adds a solid box that weighs the body's settings density. The long form adds two arguments:

* `density`: kg per cubic metre for this box, or `Float.NaN` for the settings density.
* `collides`: `false` for a mass only box. It adds weight and changes how the body turns, but never touches anything. Vehicles use this for ballast.

Boxes thinner than 0.06 m on any axis are padded to that, since Jolt rejects flat boxes. That is why carpets and pressure plates still work as bodies.

How the shape becomes a Jolt shape:

* If every box uses the settings density and collides, Jolt works out mass and inertia from the density. One box becomes a `BoxShape`; several are first merged (below) and become a static compound.
* If any box has its own density or is mass only, Zarith works out the total mass, centre of mass and full inertia tensor itself from every box, builds the shape from the colliding boxes only, shifts its centre of mass to match, and sets the mass properties on the body.
* Boxes that share a whole face are merged into one, along X, then Z, then Y, for up to four rounds. Seams between the boxes of a compound catch on anything sliding across them; a single box has none.
* Convex radius is Jolt's default or half the smallest half extent, whichever is smaller.

Other methods: `isEmpty()`, `boxCount()`, `boxes()` (each as `{centreX, centreY, centreZ, halfX, halfY, halfZ, density, collides}`), `volume()`, `mass(defaultDensity)`.

## BodySettings

Everything needed to add a body. The defaults suit one block.

| Field | Default | Meaning |
|---|---|---|
| `position` | (0, 0, 0) | Where the shape's origin goes in the world. `at(x, y, z)` sets it. |
| `rotation` | identity | `rotated(q)` sets it. |
| `linearVelocity`, `angularVelocity` | zero | `moving(linear, angular)` sets both. |
| `density` | 500 | kg per cubic metre for boxes without their own. |
| `friction` | 0.6 | 0 is frictionless, 1 grips. Combines with whatever it touches. |
| `restitution` | 0.1 | Bounciness. |
| `linearDamping` | 0.05 | |
| `angularDamping` | 0.1 | |
| `gravityFactor` | 1 | Multiplies the space's gravity for this body. |
| `awake` | `true` | `false` adds the body asleep. Groups use this to load sleeping bodies asleep. |
| `continuousCollision` | `true` | Sweeps the body each step (Jolt's `LinearCast` motion quality) so a small fast body cannot pass through a one block floor. |
| `passesSoftTerrain` | `false` | Puts the body on the `PASSING` layer so it falls through soft terrain. |

## PhysicsBody

Every method is a no-op once the body is destroyed, and reads return zero or the input unchanged. Holders never need to guard against a body that vanished because its level unloaded.

**Pose and frames**

| Method | Notes |
|---|---|
| `readPose(Vector3d, Quaternionf)` | Origin of the shape's frame and rotation. |
| `centerOfMass(Vector3d)` | World centre of mass. |
| `localCenterOfMass()` | Centre of mass in the shape's frame. Do not modify the returned vector. |
| `toWorld(local, out)`, `toWorldDirection(local, out)`, `toLocal(x, y, z, out)` | Frame conversions. |
| `bounds()` | World bounds as `{minX, minY, minZ, maxX, maxY, maxZ}`. |
| `mass()` | Kilograms. |

**Motion**

| Method | Units | Notes |
|---|---|---|
| `linearVelocity(out)`, `angularVelocity(out)` | m/s, rad/s | |
| `setLinearVelocity`, `setAngularVelocity`, `setVelocities` | m/s, rad/s | Wake the body. |
| `addVelocity(dx, dy, dz)`, `addAngularVelocity(...)` | m/s, rad/s | Changes velocity directly, ignoring mass: a pebble and a boulder move alike. Controllers use `addAngularVelocity` so they steer the same whatever the body weighs. |
| `addImpulses(linear, angular)` | N·s, N·m·s | Through the centre of mass. Heavier bodies move less. |
| `addImpulse(x, y, z, atX, atY, atZ)` | N·s at a world point | Can add spin. |
| `addForceAndTorque(fx, fy, fz, tx, ty, tz)` | N, N·m | About the centre of mass, for the next step only. **Ignored while the body is asleep.** |
| `driveTowards(x, y, z, responsiveness, maxSpeed)` | | Steers the centre of mass toward a point like a physics gun, fast when far and settling on arrival, and damps spin by 20% per call. Drag mode uses it with `setHeld(true)`. |

**State**

| Method | Notes |
|---|---|
| `setHeld(boolean)` | A held body ignores gravity so it does not sag in the player's grip. Releasing restores its gravity factor. |
| `setGravityFactor(f)` | Changes the body's own factor (remembered as its base). |
| `friction()`, `setFriction(f)` | |
| `isActive()`, `activate()`, `deactivate()` | Jolt's awake state. |
| `owner()`, `setOwner(Object)` | Whatever you tie to the body, usually the entity drawing it. |
| `joints()`, `vehicle()` | The joints holding it and its wheeled vehicle, if any. |
| `space()`, `isDestroyed()`, `destroy()` | |

### Sleeping

Jolt puts bodies to sleep when they stop moving, and a sleeping body costs almost nothing. Three things follow from that and catch people out:

1. **A sleeping body feels no gravity.** A body that falls asleep in mid-air stays there. Zarith uses this on purpose for parked hover ships.
2. **`addForceAndTorque` does nothing on a sleeping body.** Wake it with `activate()` first, or use a method that wakes it (`setLinearVelocity`, `addImpulse` and the like).
3. **Something has to wake it.** Terrain changes near a body wake it, a joint being made or removed wakes both ends, and the controllers wake their body whenever the pilot asks for anything.

Block groups add a rule of their own on top of Jolt's: a group older than 40 ticks that has crept slower than 0.12 m/s and 0.06 rad/s for 30 ticks in a row is put to sleep, unless it is held, carrying a passenger, bobbing on the swell, or freshly broken off. Without it, a body balanced on an edge can creep for a long time and keep paying for a full step each tick.

## Terrain

A space does not copy the world into Jolt. It mirrors only the blocks around its bodies, and keeps them up to date by polling.

### How a body's region is chosen

Each step, every body gets a box of blocks: its bounds plus one block on every side, stretched ahead in the direction it is moving by one step's travel (at most 12 blocks per axis). If that box would be more than 96 blocks on any axis, it is clamped to 96 around the body's centre, so a body flung at speed does not mirror a huge slab of terrain. Only the cells entering and leaving the box are touched when it moves, so a big body moving costs only its edges.

Chunks that are not loaded are left out. Zarith never loads a chunk.

### How blocks become colliders

* Each block becomes a cell holding its collision boxes, its friction and whether it is soft.
* A full cube with a full cube on all six sides can never be touched, so it gets no boxes. This keeps flat ground and piles cheap. Soft blocks never count as full cubes here, so a block buried only in leaves keeps its collider for bodies falling through them.
* Cells are grouped into 16 by 16 by 16 sections. Each section becomes one static body per surface (a distinct friction and soft pair). Full cubes in a section are greedily merged into big boxes. Per block bodies would leave seams that sliding bodies catch on.
* Terrain boxes have zero convex radius. A rounded edge dips below the face beside it and makes a notch a sliding body can catch on.
* Identical box sizes share one Jolt shape.

### How changes are noticed

Every body re-checks the cells in its region by comparing each block's state with the one it was built from. Awake bodies check every 2 ticks with a budget of 8,192 cells a tick; sleeping bodies every 20 ticks with 2,048. A region bigger than the budget is checked in slices of columns over several ticks. A changed block rebuilds its own cell and its six neighbours (a neighbour may stop being buried), marks its section for a rebuild, and wakes sleeping bodies within 1.5 blocks.

This is why Zarith needs no block change events: breaking the floor under a body, or blowing it up, is noticed within a tick or two by every body standing near it.

### Eviction

Cells are reference counted by the bodies whose regions hold them. A cell no body holds is kept for 100 ticks in case a body comes back, then removed. Eviction runs every 20 ticks and only looks at the oldest released cells, so a fast vehicle's trail costs only what drops off the end.

### TerrainSource

`TerrainSource` is the interface a space reads blocks through. Implement it to run a space against something other than a Minecraft level.

| Method | Contract |
|---|---|
| `minBuildY()`, `maxBuildY()` | Lowest Y that can hold a block, and one above the highest. |
| `isLoaded(blockX, blockZ)` | Whether the column may be read. Unloaded columns are skipped. |
| `stateToken(x, y, z)` | Any object compared with `==`. A different token means the block changed. Minecraft block states work as is, because they are interned. |
| `collisionBoxes(x, y, z, sink)` | The block's collision boxes in block local coordinates within [0, 1]. Boxes thinner than 0.002 on an axis are ignored. |
| `isFullCube(x, y, z)` | Whether the collision is exactly the full cube. |
| `isSoft(x, y, z)` | Defaults to `false`. Soft blocks stop only bodies that do not pass soft terrain. |
| `friction(x, y, z)` | Jolt friction for the block's surface, 0 to 1. |

All calls happen on the server thread. `LevelTerrain`, the Minecraft implementation, caches the last chunk it looked up for the rest of the tick, because the checks walk columns block by block.

## Blocks as bodies

`com.raiiiden.zarith.physics.BlockBodies` (in `common`) decides how a block state becomes body material.

| Property | Rule |
|---|---|
| Shape | The block's collision shape centred on the block's middle. Blocks with no collision use their outline shape. Blocks with neither become a 0.5 m cube. More than 8 boxes (a fence connected on every side, a cauldron) is replaced by the block's bounding box. Results are cached per block state. |
| Density | `250 + 350 × hardness`, clamped to 100..3000 kg/m³. Unbreakable blocks count as hardness 50. Wool is light, stone is about 775, obsidian hits the cap. |
| Friction | Vanilla slipperiness `s` becomes `(1 - s) × 1.5`, clamped to 0.02..1. Normal blocks (0.6) give 0.6, ice (0.98) gives 0.03. |
| Restitution | 0.1. |

Terrain uses the same friction rule, so a block slides on ice in the world as well.

## Using a space directly

A minimal custom body on the level's space, driven by your own code. You are responsible for destroying it.

```java
PhysicsSpace space = LevelPhysics.get(level);
if (space == null) return;   // no Jolt on this machine

BodySettings settings = new BodySettings().at(x, y, z);
settings.density = 800f;
PhysicsBody ball = space.createBody(BodyShape.box(0.25f, 0.25f, 0.25f), settings);
ball.setOwner(myEntity);

// each server tick, before LevelPhysics steps the space:
ball.readPose(position, rotation);
ball.activate();
ball.addForceAndTorque(0, liftNewtons, 0, 0, 0, 0);

// when the entity goes:
ball.destroy();
```

If your object should also be walked on, shot and sheltered under, implement `ZarithSolid` on its entity. See [Solids API](solids-api.md).
