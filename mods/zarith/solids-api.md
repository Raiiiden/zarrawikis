# Solids API

[Back to Zarith](../zarith.md)

`com.raiiiden.zarith.api.ZarithSolid` is for anything that moves and should behave like blocks: entities stand on it, arrows stick in it, explosions are blocked by it, rain stops on it. Block groups implement it, and so do Felled's fallen trees. Implement it yourself when you have a moving object that is not a block group, such as an entity with its own model and a few collision boxes.

The point of the interface is that you write no mixins. Zarith has one set of mixins for all of this, and every registered solid gets their effects. Two mods redirecting the same vanilla call crash the game, so if your mod and Zarith both mixed into entity collision, players would get a crash on launch.

## What you get

| Feature | Vanilla hook | Side |
|---|---|---|
| Entities collide with your boxes, walk up slopes up to 50°, step down slopes, slide off steep faces. | `Entity.collideBoundingBox`, `Entity.collide` | Server for non players, client for the local player |
| Entities report landings to you (`landed`). | `Entity.collide` | Same |
| Sneaking stops at your edges. | `Player.canFallAtLeast` | Both |
| The local player is carried by you and pushed out of you before it moves. | `Player.tick` | Client |
| Standing on you is not "floating" to the server's anti cheat. | `ServerGamePacketListenerImpl.noBlocksAround` | Server |
| Arrows and other projectiles hit you. | `ProjectileUtil.getHitResult`, `AbstractArrow.tick`, `Projectile.hitTargetOrDeflectSelf` (26.3) | Both |
| Explosions see your blast resistance and cannot see through you. | `ServerExplosion` | Server |
| Rain does not fall under you; snow, ice and cauldrons skip your columns. | `Level.isRainingAt`, `ServerLevel.tickPrecipitation` | Server |
| Rain and snow are drawn stopping on your top, and splash there. | Weather rendering | Client |
| Sky light and ambient occlusion darken under and beside you. | `getSkyLightLevel`, light coords, `getShadeBrightness` | Client |

## Registering

Register the solid when it joins a level and remove it when it leaves, **on both sides**:

```java
ZarithSolids.add(this);     // when it joins a level
ZarithSolids.remove(this);  // when it leaves
```

For an entity, the reliable place is the level's own entity callback, which fires for every way an entity can arrive or go, chunk loading and unloading included. `BlockGroupEntity` wraps `setLevelCallback` for this; copy that approach.

The set of solids is stored on the `Level` object itself (a mixin adds a `SolidIndex` field), so it goes away with the level and nothing can leak across worlds. Solids that report `isSolidRemoved()` are dropped from the set the next time it is searched.

### Lookups

| Method | Notes |
|---|---|
| `ZarithSolids.in(level)` | Every solid in the level, unmodifiable. |
| `ZarithSolids.any(level)` | Cheap check before doing any work. Every Zarith mixin calls this first, so a world with no solids costs almost nothing. |
| `ZarithSolids.near(level, area)` | Live solids whose bounds meet the box. |
| `ZarithSolids.coverTop(level, column)` | Highest solid top over a column, or `NaN`. |

With 12 or fewer solids in a level, `near` just checks them all. With more, it uses a grid of 8 by 8 block columns rebuilt at most once a tick, with each solid's bounds padded by 2 blocks so one that moved since the rebuild is still found. A search covering more than 64 columns falls back to checking every solid.

## The interface

Methods are named `solidLevel`, `isSolidRemoved` and `solidBounds` rather than reusing `Entity`'s `level()`, `isRemoved()` and `getBoundingBox()`. On obfuscated Minecraft versions those entity methods are renamed in the shipped jar, and an interface method an entity only inherits would then have no implementation at run time.

**Identity and shape**

| Method | Contract |
|---|---|
| `Level solidLevel()` | The level it is in. |
| `boolean isSolidRemoved()` | `true` once it is gone. |
| `AABB solidBounds()` | World box around everything it fills. Kept up to date; every search tests it first. |
| `boolean isSolid()` | Whether entities collide with it. Projectiles, blasts and weather see it either way. |
| `boolean ignores(Entity)` | Entities it never collides with. Default `false`. Block groups ignore the player holding them and their own passengers. |

**Collision**

| Method | Contract |
|---|---|
| `double sweep(AABB box, Axis axis, double move)` | How far `box` can move along `axis` (signed, `move` is the wish) before it touches you. Return `move` when nothing is in the way. After a hit, `sweepNormalX/Y/Z()` must return the normal of the face that stopped it; its Y decides whether the face is walkable (at least `cos 50°`). |
| `double sweepNormalX()`, `Y()`, `Z()` | The normal from the last `sweep`. |
| `boolean deepens(AABB from, AABB to)` | Whether moving a box from `from` to `to` pushes it further into any of your boxes than it already was. Used to reject vanilla's step up when it lifts a box into an overhang. |
| `boolean touches(AABB area)` | Solid and overlapping the area. |
| `boolean overlaps(AABB area)` | Overlapping the area whether solid or not. |
| `double clip(Vec3 from, Vec3 to, Vector3d hitNormal)` | Fraction along the segment where it first enters you, or `NaN`. Put the face normal in `hitNormal`. A segment starting inside counts as a hit at 0. |
| `float resistanceAt(x, y, z)` | Blast resistance of whatever fills that point, or -1. |

**Cover and shade**

| Method | Contract |
|---|---|
| `double coverTop(int x, int z)` | Highest point of you over the block column, or `NaN`. Called a lot; results are cached per tick by the index. |
| `Long2DoubleOpenHashMap cover()` | `coverTop` for every column you cover, keyed by `ChunkPos.pack(x, z)` (`asLong` on older versions). |
| `double topAt(double x, double z)` | Exact top at a point, or `NaN`. Rain splashes land here. |
| `Long2DoubleOpenHashMap shadeTops()` | Top at the centre of each column, for the client's sky light. Default derives it from `cover()` and `topAt`. |
| `LongSet shadeCells()` | World block spaces you fill, as `BlockPos` longs, for ambient occlusion. Default empty. |
| `boolean castsShade()` | Default `true`. Return `false` while moving: every change re-meshes the chunks under you. |

**Entities**

| Method | Contract |
|---|---|
| `void landed(Entity entity, float speed)` | An entity landed on you at `speed` m/s. Called on the server for non player entities and on the client for the local player. If you want players to push you, send the client call to the server yourself. |
| `void pushOut(Entity entity, Vec3 carried)` | You may have moved into the entity: move it out. `carried` is how far it was already carried this tick. Called on the client for the local player each tick. On the server it is up to you to call it for other entities. |
| `Vec3 clientAdvance(Player player)` | Client only. Step to this tick's pose and return how far that moves the point under `player`'s feet if the player stands on you, else `null`. Called at the start of every client tick for every solid, before any entity moves. |

The server side of carrying and pushing other entities is not done for you. `BlockGroupEntity` does it in its own tick: it finds riders before reading the new pose, carries them by the distance under their feet, then calls `pushOut` for every nearby non player entity. A custom solid that moves should do the same.

## Helper classes

### SolidFrame

`SolidFrame` does the geometry for boxes that all share one rotation, which is the common case: every box of a group turns with the group. It is a separating axis test between an axis aligned entity box and a turned box, in double precision.

```java
SolidFrame frame = new SolidFrame();
frame.set(rotation);          // once per pose change
```

Then for each of your boxes, given its world centre `(bx, by, bz)` and half extents `(hx, hy, hz)` along your own axes:

| Method | Returns |
|---|---|
| `contact(bx, by, bz, hx, hy, hz, box, dx, dy, dz)` | Distance `box` can travel along the unit direction before touching, or `SolidFrame.NONE`. Sets `frame.nx/ny/nz` to the contact normal; ties pick the more upward one. |
| `depth(bx, by, bz, hx, hy, hz, box)` | How deep `box` is inside, 0 or less when apart. |
| `escape(...)` | The shortest way out, normal in `nx/ny/nz`. |
| `exitAlong(..., box, dx, dy, dz)` | Distance to move along a direction to get out. |
| `rayHit(..., ox, oy, oz, dx, dy, dz)` | Where the segment `o + t × d` (t from 0 to 1) enters, or `NONE`. Starting inside counts as `t = 0`. |
| `columnSpan(..., x, z, out)` | The vertical extent over a world point into `out[0..1]`; `false` if the column misses. |
| `contains(..., x, y, z)` | Whether the point is inside. |
| `halfExtent(i, hx, hy, hz)` | Half size of the turned box along world axis `i`, for a coarse bounding box. |
| `toWorld(ox, oy, oz, lx, ly, lz, out)` | A local point to world. |

`SolidFrame.EPS` is `1e-7`. `SolidFrame.SHALLOW` is `0.05`: a box deeper than this inside a solid is treated as stuck and left alone, as vanilla does with blocks.

### SolidCollision

Static helpers around the collision mixin.

| Method | Use |
|---|---|
| `againstBlocks(entity, move)` | Moves an entity's box against world blocks only. Use it when carrying an entity, so you never carry it into a wall. |
| `clearOfBlocks(entity, from, to)` | Whether a box can move without entering a world block it was not already in. |
| `touches(entity, level, area)` | Whether any solid the entity collides with touches the area. |
| `standingOnSolid(player)` | Whether the player stands on a solid. |
| `nearSolid(entity)` | Standing on or right beside a solid. |
| `collides(entity, level)` | Whether this entity collides with solids at all here: not spectators, not `noPhysics`, not other solids, and not players on the server. |
| `WALKABLE` | `cos 50°`. |

### SolidCover

What weather, projectiles and explosions call. You can call it too.

| Method | Use |
|---|---|
| `top(level, x, z)` | Highest solid over a column. |
| `shelters(level, pos)` | Whether a solid covers the position (top more than half a block above it). |
| `rainHeight(level, x, z, vanilla)` | Where rain stops. |
| `overlaps(level, area)` | Whether any solid fills part of the box. |
| `resistanceAt(level, x, y, z)` | Blast resistance of the solid filling a point, or -1. |
| `clip(level, from, to, blockHit)` | The earlier of a block hit and the first solid hit along the segment, as a `BlockHitResult`. |

A hit on a solid is reported as a `BlockHitResult` whose block position is the air block just inside the hit point. Projectile code that checks `getBlockState(hit.getBlockPos())` sees air there, which is why Zarith also patches the arrow and projectile hit code.

### SolidMotion

Called from Zarith's client mixins: `advanceAll(level, player)` at the start of the client tick and `beforeLocalPlayerTick(player)` at the start of the local player's tick. You do not call these.

## A minimal solid

A sketch of a moving platform entity with one box, to show the shape of an implementation. It skips pose syncing and server side carrying, which you would add as `BlockGroupEntity` does.

```java
public class PlatformEntity extends Entity implements ZarithSolid {

    private static final double HX = 2, HY = 0.25, HZ = 2;
    private final Quaternionf rotation = new Quaternionf();
    private final SolidFrame frame = new SolidFrame();
    private double nx, ny = 1, nz;

    // ... constructor, ticking, syncing ...

    @Override
    public void setLevelCallback(EntityInLevelCallback callback) {
        super.setLevelCallback(callback == EntityInLevelCallback.NULL ? callback : new EntityInLevelCallback() {
            @Override public void onMove() { callback.onMove(); }
            @Override public void onRemove(RemovalReason reason) {
                callback.onRemove(reason);
                ZarithSolids.remove(PlatformEntity.this);
            }
        });
        if (callback != EntityInLevelCallback.NULL) ZarithSolids.add(this);
    }

    @Override public Level solidLevel() { return level(); }
    @Override public boolean isSolidRemoved() { return isRemoved(); }
    @Override public AABB solidBounds() { return getBoundingBox(); }
    @Override public boolean isSolid() { return true; }

    @Override
    public double sweep(AABB box, Direction.Axis axis, double move) {
        nx = 0; ny = 1; nz = 0;
        double dx = axis == Direction.Axis.X ? Math.signum(move) : 0;
        double dy = axis == Direction.Axis.Y ? Math.signum(move) : 0;
        double dz = axis == Direction.Axis.Z ? Math.signum(move) : 0;
        double t = frame.contact(getX(), getY(), getZ(), HX, HY, HZ, box, dx, dy, dz);
        if (t == SolidFrame.NONE || t > Math.abs(move)) return move;
        nx = frame.nx; ny = frame.ny; nz = frame.nz;
        return Math.copySign(t, move);
    }

    @Override public double sweepNormalX() { return nx; }
    @Override public double sweepNormalY() { return ny; }
    @Override public double sweepNormalZ() { return nz; }

    // deepens, touches, overlaps, clip, resistanceAt, coverTop, cover, topAt,
    // landed, pushOut and clientAdvance follow the same pattern with the
    // matching SolidFrame method.
}
```

Call `frame.set(rotation)` whenever the rotation changes, and keep `solidBounds()` covering the turned box.
