# Block groups

[Back to Zarith](../zarith.md)

A block group is an entity holding any number of blocks that move together as one rigid body. One physics block, a rope link, a felled tree, an assembled house and a car are all block groups. The base class is `com.raiiiden.zarith.group.BlockGroupEntity`.

Out of the box a group:

* collides with terrain and with other bodies,
* lets entities stand on it, carries them, and pushes them out of its way (if it is solid to entities),
* floats, drags and drifts in water and lava (if its buoyancy hook says so),
* catches fire per block, spreads it, and burns blocks away,
* can be mined block by block, stripped with an axe and lit with flint and steel,
* is shoved and broken by explosions,
* thuds, kicks up dust and crushes entities it falls on,
* holds arrows that hit it,
* splits into separate groups when a block holding it together goes,
* saves and loads, and syncs to clients.

[World interaction](world-interaction.md) and [Destruction](destruction.md) cover those behaviours in detail. This page covers the data structures and how to make and customise groups.

## PhysicsBlockEntity

`com.raiiiden.zarith.entity.PhysicsBlockEntity` is Zarith's own concrete group, registered as `zarith:physics_block`. Every group the `/zarith` commands make is one of these. Use it when you want a group with standard behaviour and do not need your own entity type.

It adds a flags byte, synced to clients and saved as `Flags`:

| Flag | Value | Effect |
|---|---|---|
| `PhysicsBlockEntity.COLLISION` | 1 | Entities stand on it and bump into it. Without it they pass through, though it still hits terrain and bodies. |
| `PhysicsBlockEntity.FLOATS` | 2 | Floats on water and lava, drifts with currents at 1.5 m/s and rocks on a swell. Every block floats settled 0.6 deep whatever it is made of. Without it the group still feels fluid drag but sinks. |

It also turns load bearing on, so physics blocks take part in [structural collapse](destruction.md).

```java
// One block centred on a point
PhysicsBlockEntity.create(level, state, centre, flags);

// One block centred on a point, turned about its centre
PhysicsBlockEntity.create(level, state, centre, rotation, flags);

// Any grid; origin is where the grid's (0, 0, 0) corner goes
PhysicsBlockEntity.create(level, grid, origin, rotation, flags);
```

Each `create` only builds the entity. Call `level.addFreshEntity(entity)` to spawn it, and call `setVehicle` or `link` before that if you need them.

## BlockGrid

`com.raiiiden.zarith.group.BlockGrid` stores a group's blocks on an integer grid.

### Layout

Blocks are kept in 8 by 8 by 8 bricks. Only bricks that hold a block exist. Each brick has four 512 bit masks (a block is here, it is live, it collides, it weighs) and a palette of block states with a byte or short index per spot, or no index array at all while the brick holds a single state. A brick of one material costs a few hundred bytes whatever its size.

### Block ids

A block is addressed by an `int` id: its brick's slot times 512 plus its spot inside the brick. **Ids never change.** A removed block keeps its id, its position and its state, and `restore(id)` brings it back. That is why packets, saved references and client predictions can all use ids safely. It also means an id can name a block that is no longer there, so check `isLive(id)` before using one you kept.

```java
for (int id = grid.firstLive(); id >= 0; id = grid.nextLive(id)) {
    BlockState state = grid.state(id);
    CellKind kind = grid.kind(id);
    int x = grid.x(id), y = grid.y(id), z = grid.z(id);
}
```

### Methods

| Method | Notes |
|---|---|
| `add(x, y, z, state, kind)` | Returns the id. Adding to an occupied position returns the existing id and changes nothing. |
| `remove(id)`, `restore(id)` | Return whether anything changed. Removing a solid records its position as cut (see below). |
| `setState(id, state)` | Changes the block in place. |
| `exists(id)` | The id names a block, live or removed. |
| `isLive(id)` | The block is there now. |
| `x(id)`, `y(id)`, `z(id)` | Grid position of the block's minimum corner. |
| `cx(id)`, `cy(id)`, `cz(id)` | Its centre (`x + 0.5` and so on). |
| `pos(id)` | Position packed as `BlockPos.asLong`. |
| `state(id)`, `kind(id)` | For any id that exists, live or not. |
| `liveAt(x, y, z)` | Id of the live block there, or `BlockGrid.NONE` (-1). |
| `firstLive()`, `nextLive(id)`, `forEachLive`, `liveIds()` | Iterate live blocks in id order. |
| `liveCount()`, `liveSolidCount()` | |
| `bounds()` | Extent of live blocks as `{minX, minY, minZ, maxX, maxY, maxZ}`, max exclusive. Returns a copy. |
| `idLimit()` | Every id is below this. Size arrays indexed by id with it. |
| `version()` | Increases on every change. Use it to tell whether cached data is stale. |
| `mergedBoxes(filter, density)` | Live, weighing blocks merged into as few boxes as possible, only joining blocks of the same kind and density. |
| `components()`, `components(cracks)` | Connected parts (below). |
| `subset(ids)` | A new grid of these blocks at the same positions. |
| `write(buf)`, `read(buf)`, `save()`, `load(tag, level)` | Network and NBT. |

### Connectivity

`components()` returns the group's connected parts as lists of ids. The rules:

* Solid blocks join any of their 26 neighbours that are also solid, diagonals included.
* A diagonal bond holds only while it is as built. If a solid block that sat between the two diagonal blocks was ever removed (its position is in the grid's cut set), the diagonal bond is broken. This stops two blocks touching at an edge from holding a structure together after the block that really joined them is gone.
* Non solid blocks (`SOFT`, `LOOSE`) hang on the nearest solid part through faces, spreading outward from the solids. Non solid blocks that reach no solid form parts of their own.
* `components(cracks)` also treats the listed solid pairs (`BlockGrid.crackKey(a, b)`) as broken.

The cut set is saved with the grid and carried into pieces by `subset`, so a diagonal that was cut stays cut after the group splits or reloads.

## CellKind

Every block in a group has a kind, decided when it is added.

| Kind | Collides | Weighs | Typical blocks |
|---|---|---|---|
| `SOLID` | yes | yes | Logs, stone, planks, anything with a collision shape. |
| `SOFT` | no | yes | Leaves on a felled tree. |
| `LOOSE` | no | no | Vines, cocoa, snow layers, torches: things that ride along. |

`SOFT` blocks still count for mass and inertia but not for collision, so a tree's crown makes it top heavy without snagging on everything. A group whose `softBlocksEntities()` returns `true` lets entities stand on its `SOFT` blocks too, though they still pass through terrain and bodies.

`/zarith assemble` makes blocks with a collision shape `SOLID` and everything else `LOOSE`.

The ordinals are saved, so new kinds must be added at the end of the enum.

## How the body's shape is built

Each time the body is built or reshaped, the group turns its grid into a `BodyShape`:

1. Live full cube blocks that weigh are merged into boxes with `mergedBoxes`, joining only blocks of the same kind and the same density. Mass only `SOFT` boxes and colliding `SOLID` boxes stay separate.
2. Every other weighing block adds its own collision boxes from `BlockBodies.shapeOf`, at its own density.
3. A vehicle's ballast, if any, is added as one mass only box.

Each block's density comes from the `cellDensity(cell)` hook, which by default is `BlockBodies.density` for weighing blocks and 0 for `LOOSE` ones. Blocks with zero density are left out.

After blocks are removed, or a block's state changes, the body is reshaped in place. Its joints and vehicle are rebuilt about the new centre of mass by the core.

## Making your own group type

Extend `BlockGroupEntity` when you want behaviour of your own: a tree that leaves saplings, a ship hull with its own drag, a wrecking ball.

```java
public class CrateEntity extends BlockGroupEntity {

    public CrateEntity(EntityType<? extends CrateEntity> type, Level level) {
        super(type, level);
    }

    public static CrateEntity create(ServerLevel level, BlockGrid grid, Vector3d origin) {
        CrateEntity crate = new CrateEntity(MyEntities.CRATE.get(), level);
        crate.setup(grid, origin, new Quaternionf(), new Vector3f(), new Vector3f());
        return crate;
    }

    // Pieces that break off are crates too.
    @Override
    protected BlockGroupEntity newPiece(ServerLevel level) {
        return new CrateEntity(MyEntities.CRATE.get(), level);
    }

    @Override
    protected float buoyancy(int cell) {
        return cellDensity(cell) / 0.5f;   // floats half under
    }

    @Override
    protected boolean fireSpreadsToWorld() {
        return false;
    }
}
```

What you have to do yourself:

1. **Register the entity type** the way your loader wants. Copy Zarith's builder settings: `MobCategory.MISC`, `fireImmune()`, a client tracking range of 10 chunks, and **an update interval of 1**. The server sends poses itself every tick, but vanilla's own tracking packets should not lag behind. The size you give the builder does not matter, because the group overrides its bounding box.
2. **Register the renderer.** `BlockGroupRenderer` is generic over any `BlockGroupEntity`, so pass its constructor: `BlockGroupRenderer::new`.
3. **Call `setup` before spawning.** `setup(grid, origin, rotation, linearVelocity, angularVelocity)` fills a new group. Spawn with `level.addFreshEntity`.
4. **Implement `newPiece`.** It returns an empty group of your type for pieces that break off. Return `null` to drop pieces instead of spawning them; the blocks then stay in the parent.

On 1.20.1 and 1.21.1, extra saved data goes in `readAdditionalSaveData(CompoundTag)` and `addAdditionalSaveData(CompoundTag)`; call `super` first. On 26.x those methods take `ValueInput` and `ValueOutput`, and you override `readGroup(CompoundTag)` and `writeGroup(CompoundTag)` instead, which hold the whole group under one key (`saveKey()`, default `"zarith"`).

## Hooks

Override these to tune a group. All are `protected` unless marked. "Server" hooks run on the logical server only.

### Body and material

| Hook | Default | Notes |
|---|---|---|
| `cellDensity(int cell)` | `BlockBodies.density(state)` if the block weighs, else 0 | Kilograms per full block. |
| `configureBody(BodySettings)` | Friction of the first solid block, restitution 0.1 | Set friction, damping, gravity factor, and `passesSoftTerrain` here. Called each time the body is built. |
| `wantsSolid()` | `true` | Whether entities collide with the group. Asked every server tick; the answer is synced. |
| `softBlocksEntities()` | `false` | Whether `SOFT` blocks also stop entities. Sent to clients when the group spawns for them. |

### Fluids

| Hook | Default | Notes |
|---|---|---|
| `buoyancy(int cell)` | 0 | Kilograms of fluid one full block displaces when under. Return `cellDensity(cell) / d` to float settled `d` deep. 0 sinks. |
| `fluidDrag(int cell, boolean lava)` | `cellDensity × 0.5` in water, `× 2.1` in lava | Newtons per m/s for one full block. |
| `currentSpeed()` | 0 | m/s a flowing fluid pushes the group at. |
| `swayAmount()` | 0 | Strength of the swell in still water. |
| `riderWeightShare()` | 1 | How much of the weight of entities standing on a floating group pushes it down. |

### Entities

| Hook | Default | Notes |
|---|---|---|
| `landingShare()` | 1 | Share of a landing entity's impact pushed into the group. |
| `pushImpulse(float share)` | Grows with mass, capped | Impulse a walking push gives, for a pusher `share` players heavy. |
| `crushes()` | `true` | Whether a falling group hurts entities under it. |
| `crushDamage(float closing)` | Up to 20, scaled by mass | Damage for a closing speed in m/s. |
| `crushDamageType()` | `zarith:physics_block` | |
| `crushAttacker(ServerLevel)` | Whoever last held or threw it | The player credited with crush kills. |

### Fire

| Hook | Default | Notes |
|---|---|---|
| `burns()` | `true` | `false` puts out and refuses all fire. |
| `fireSpreadsToWorld()` | `true` | Whether fire on the group can light world blocks. |
| `flammable(int cell)` | Vanilla burn odds above 0 | |
| `smokeAmount()` (public, client) | 1 | Multiplier on smoke particles. |

### Explosions and tools

| Hook | Default | Notes |
|---|---|---|
| `explosionsAffect()` | `true` | `false` ignores blasts. |
| `blastBreaks(cell, distance, radius)` | Vanilla style damage vs. blast resistance, with chance | Whether a blast breaks the block loose. |
| `blastWear(cell, distance, radius)` | Half the breaking share | Wear added to a block that did not break (load bearing groups only). |
| `canStrip(int cell)` | `true` | Whether an axe may strip, scrape or unwax it. |

### Load and destruction

| Hook | Default | Notes |
|---|---|---|
| `bearsLoad()` | `false` (`true` on `PhysicsBlockEntity`) | Turns on structural collapse, blast wear and impact breaks. |
| `bondStrength(int cell)` | Hardness clamped to 0.2..10, divided by 1.5; unbreakable 1000 | 1 is about stone. |
| `onBondsBroken(level, from, to)` | Block dust, smoke and a break sound at up to 8 bonds | Cells in pairs. |
| `looseComponent(level, ids)` | Spawns it as a piece, returns `true` | A part with no solid block broke off. Return `false` to keep it in this group. |
| `noSolidsLeft(level)` | Breaks the rest with drops and discards | |
| `onCellsRemoved(level, ids)` | nothing | Ids are already out of the grid. |
| `onPieceSpawned(piece)` | nothing | Called before the piece is added to the level. |

### Impacts and sound

| Hook | Default | Notes |
|---|---|---|
| `impactSound(heavy, state)` | Place sound when heavy, hit sound otherwise | |
| `impactSize()` | `mass / 700`, clamped 0.4..1.6 | Louder and deeper for bigger groups. |
| `impactDust(level, impact, x, y, z)` | Block particles | Not called for hits under water. |
| `onImpacts(level, impacts)` | nothing | All of this tick's impacts, hardest first. |

### Lifecycle

| Hook | When |
|---|---|
| `onJoinedLevel()` | The group was added to a level, either side. |
| `onLeftLevel()` | It left, for any reason, either side. |
| `onPose(level)` | Server, right after the pose was read from the body. |
| `onAwake(level)` | Server, each tick the body is awake, after impacts. |
| `onSettle(level, awake)` | Server, every tick before fluids and fire. |
| `onEndTick(level)` | Server, last thing each tick, after the pose was sent. |
| `cullsInnerSoft()` (public, client) | Whether faces between two `SOFT` blocks are left out of the mesh. Default `false`. |

Several hooks can remove the group (by burning, breaking or splitting it). Code after a hook in your own override should check `isRemoved()`.

## Public methods

**State**

| Method | Notes |
|---|---|
| `grid()` | The `BlockGrid`. Do not add or remove blocks on a live group directly; use `removeCells` (subclasses), `carve`, `mineCell` or `setCellState`, which keep the body, clients and fire in step. |
| `body()` | The `PhysicsBody`, or `null` on the client and before the first server tick. |
| `ensureBody(ServerLevel)` | Makes the body now if there is none. Used when another group's joint needs it this tick. |
| `mass()` | Body mass, or summed from the blocks before there is a body. |
| `currentOrigin()`, `currentRotation()` | The pose as of this tick. Do not modify them. |
| `centerVec()` | World centre of mass. |
| `holderId()`, `setHolder(player)` | The player holding it in drag mode, or -1. |
| `rememberThrower(player)` | Credits crush kills to this player for the next 200 ticks. |
| `isDebris()`, `setDebris(boolean)` | See [debris](destruction.md#debris). |

**Frames**

| Method | Notes |
|---|---|
| `localToWorld(x, y, z, out)`, `worldToLocal(x, y, z, out)` | Grid frame to world and back. |
| `cellToWorld(cell, out)` | World centre of a block. |
| `cellAt(x, y, z)` | Live block at a world point, or -1. |
| `worldDirection(x, y, z)` | A grid direction in world space. |
| `spotToWorld(spot, out)`, `beside(spot, direction)` | For grid positions packed as longs (fire spots). |
| `gridFrame(rotation)` (static) | For each world `Direction` (by ordinal), the grid direction closest to it. Always a proper rotation. Fire uses it so "up" in the grid follows the world's up as the group tumbles. |
| `renderPose(partialTick, origin, rotation)` | The pose to draw this frame. |

**Actions** (server)

| Method | Notes |
|---|---|
| `link(other, JointSettings)`, `unlinkAll()`, `joints()` | See [Joints](joints.md). |
| `setVehicle(GroupVehicle)`, `vehicle()` | See [Vehicles](vehicles.md). Set it before spawning. |
| `carve(level, box)` | Removes blocks whose centres are in a world box, without drops. Returns how many. |
| `setCellState(level, cell, state)` | Changes one block in place, reshapes, syncs. |
| `ignite(cell)`, `ignite(cell, toward, again)`, `igniteNear(point, radius)`, `onFire()`, `onFire(cell)`, `fires()` | See [Fire](world-interaction.md#fire). |
| `explode(centre, radius, incendiary)`, `onExplosion(level, centre, radius, incendiary)` (static) | See [Explosions](world-interaction.md#explosions). |
| `wearOf(cell)`, `wearDown(cell, amount)`, `checkStructureSoon()` | See [Destruction](destruction.md). |
| `nudge(vx, vy, spin)` | Adds velocity along X and Y and spin about Y. |
| `mineCell`, `useOnCell`, `putOutFire`, `push`, `landed`, `vehicleInput` | Called from packets. You rarely call them. |

**Global tuning** (public static fields, apply to every group)

| Field | Default | Meaning |
|---|---|---|
| `debrisMaxBlocks` | 3 | Broken off pieces this small or smaller are debris. 0 turns debris cleanup off. |
| `debrisLifetimeTicks` | 1200 | Ticks debris rests before crumbling. |
| `impactBreakSpeed` | 7.0 | m/s above which load bearing groups break where they hit. |
| `maxStructureBlocks` | 60000 | Groups with more solid blocks skip structural checks. |

## Removing blocks and splitting

Every way a block leaves a group (mining, burning, blasts, impacts, carving, collapse) ends in the same steps:

1. The blocks are removed from the grid.
2. `grid.components()` finds the parts that are left.
3. If there is more than one part, they are sorted by solid block count. The biggest stays in this group. Each other part with a solid block becomes a new group through `newPiece` and `setup`, starting with this group's pose, velocity and spin and carrying its fire. A part with no solid block goes to `looseComponent`, which by default spawns it too.
4. Fire spots that no longer touch a block go out. `onCellsRemoved` runs. Load bearing groups ask for a structural check. Clients get the removed ids.
5. The body is reshaped, or, with no blocks left at all, the group is discarded. With only non solid blocks left, the next tick calls `noSolidsLeft`.

Pieces that break off through blasts, collapse or impacts are also flagged as debris when they are small enough, and are made slippery for their first 40 ticks (friction 0.05) so they slide away from what they broke off instead of wedging against it.

## The client copy

On the client a group has the same grid, kept in step by packets, but no body. Its pose comes from buffered server snapshots drawn two ticks behind. When the local player breaks a block, the block disappears at once and comes back 30 ticks later if the server never confirmed it. See [Networking and client](networking-and-client.md).

The field `renderCache` (public, client) holds the renderer's mesh for the group. Leave it alone.
