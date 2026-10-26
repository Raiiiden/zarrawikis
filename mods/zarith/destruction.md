# Destruction

[Back to Zarith](../zarith.md)

Groups that bear load come apart the way buildings do. A wall with its base blown out drops what it held up, a balcony with its supports mined away falls, a floor that falls onto another breaks it. This page explains how that works and how to tune it.

Load bearing is off by default on `BlockGroupEntity` and on for `PhysicsBlockEntity`, which is what `/zarith physicify` and `/zarith assemble` make. Turn it on for your own group type by overriding `bearsLoad()` to return `true`. Everything on this page except splitting and `carve` needs it.

## Splitting

Splitting happens for every group, load bearing or not. When blocks leave a group, the blocks left are checked for connectivity and every part that is no longer joined to the biggest one flies off as a new group. The rules for what counts as joined are on the [Block groups](block-groups.md#connectivity) page. In short: solids join their 26 neighbours, a diagonal join only holds while the blocks that bridged it are still there, and non solid blocks hang on the nearest solid.

Splitting only cares whether blocks touch. A tower joined to a building by a single block stays one group. Whether that one block can really hold the tower is the job of the structural check.

## Structural check

`com.raiiiden.zarith.core.structure.StructureSolver` works out which blocks can no longer hold up what rests on them. It lives in `core` and knows nothing about Minecraft; the group feeds it grid positions, masses and strengths.

### When it runs

A load bearing group asks for a check:

* the tick after any of its blocks were removed,
* when its body falls asleep, so a group that just landed is judged where it rests,
* when you call `checkStructureSoon()`.

The group copies its solid blocks into a solver and runs `solve` on a single background thread named `Zarith structure`. When the answer comes back, it is applied on the server thread, unless the grid changed in the meantime. Then the answer is thrown away and a new check is asked for.

Two conditions skip the check entirely:

* The group's grid up is more than about 25° from the world's up. A group lying on its side or tumbling is not judged.
* The group has more solid blocks than `BlockGroupEntity.maxStructureBlocks` (60,000).

### Supports

A block is a support when no solid block of the group is under it in grid terms and, a quarter block below its bottom face in the world, there is either a world block with a collision shape or another solid. Supports carry any load. If no block is a support, nothing is judged: the group is falling, or resting on something the check cannot see.

### How load flows

1. Each block gets a distance from the nearest support, walking over bonds between face neighbours. Stepping up onto a block costs 1, sideways costs 2 and down (hanging) costs 3. So the shortest way down for weight prefers going straight down, then across, and hangs from above only when it must.
2. Blocks are visited from farthest to nearest. Each block's load (its own mass plus whatever flowed into it) is split among its bonds to neighbours that are closer to a support. A bond straight down gets 4 shares, sideways and up 1 share each.
3. Each bond's share is compared with what it can bear: the weaker of the two blocks' strengths times a capacity for that direction. If the share is more than that, the bond breaks.
4. A broken bond's weight finds another way down in the next round, which may break that bond too. This repeats until nothing breaks, for at most 40 rounds.
5. Everything still joined to a support through unbroken bonds stands. The rest falls.
6. A part that fell but still sits on standing blocks puts its weight on them, spread evenly over the blocks right under it, and the whole thing is weighed again. A broken off chunk resting on a ledge loads that ledge. This repeats up to 6 times.

### Capacities

The capacities are public static fields on `StructureSolver`, in kilograms carried by one bond between two blocks of strength 1:

| Field | Default | Bond direction |
|---|---|---|
| `compressionCapacity` | 2,000,000 | Weight resting on the block below. |
| `spanCapacity` | 40,000 | Weight passed sideways. |
| `hangCapacity` | 15,000 | Weight hanging from the block above. |

A block's strength comes from the group's `bondStrength(cell)` hook. By default it is the block's hardness clamped to 0.2..10, divided by 1.5:

| Block | Hardness | Strength |
|---|---|---|
| Glass | 0.3 | 0.2 |
| Planks | 2 | 1.33 |
| Stone | 1.5 | 1 |
| Obsidian | 50 | 6.67 |
| Unbreakable | -1 | 1000 |

As a sense of scale: stone weighs about 775 kg a block, so one stone to stone bond can hang roughly 19 stone blocks, pass about 50 sideways, and hold up about 2,500 stacked on it. Real structures split their load over several bonds, so they hold more than these single bond numbers.

### What happens when it collapses

`onBondsBroken(level, from, to)` runs first with the broken bonds as pairs of cells. By default it puts block dust and smoke at up to eight of them and plays a break sound at up to three.

Then each falling part becomes a new group through `newPiece`, so a collapse can drop several separate chunks. If the falling part is the whole group, no piece is made; the group's own body is woken and drops. The group that is left is not weighed again, because it was already weighed without what fell.

### Using the solver on its own

```java
StructureSolver solver = new StructureSolver();
int a = solver.add(0, 0, 0, 775f, 1f);   // x, y, z, mass in kg, strength
int b = solver.add(0, 1, 0, 775f, 1f);
int c = solver.add(1, 1, 0, 775f, 1f);
solver.setSupport(a, true);

StructureSolver.Result result = solver.solve(0, 1, 0);   // up is +Y
if (result.collapses()) {
    for (IntArrayList part : result.parts()) {
        // each part falls as one piece
    }
}
```

`solve(upX, upY, upZ)` takes the structure's up as one of the six unit grid axes. `Result` holds `falling` (every block that falls), `parts` (the same blocks split into pieces held together by unbroken bonds), `brokenFrom` and `brokenTo` (the broken bonds), and `anySupport`. The solver is not thread safe, but separate solvers can run on separate threads.

## Wear

Load bearing blocks can be damaged without breaking. Each block has a wear value from 0 to 1, and reaching 1 breaks it.

* `wearOf(cell)` reads it.
* `wearDown(cell, amount)` adds to it and returns `true` when that breaks the block. It does not remove the block; the caller does.

Blasts and hard impacts use wear (below). Wear is not saved and not shown on the client.

## Blast damage

Any block a [vanilla explosion](world-interaction.md#explosions) does not break is worn by `blastWear(cell, distance, radius)`. By default that is half of the share of the block's breaking threshold the blast reached, so two blasts that each fell just short break a block together.

## Impact damage

When a load bearing group with at least two solid blocks hits something at more than `impactBreakSpeed` (7 m/s by default), the hit wears blocks on both sides:

* The amount is `(speed - 7) / 6 × clamp(mass / 2500, 0.5, 4)`, divided by each block's bond strength. Heavier groups hit harder.
* The group's own block at the point of impact takes it.
* If another load bearing group is right in front of that point, its block takes it too. A falling floor breaks the floor below.
* At most 4 blocks per group break from impacts in one tick.

The broken blocks are removed (with dust, without drops) when the group settles at the end of its tick, and then the structural check takes over.

## Debris

Pieces broken off by blasts, collapse or impact damage are made slippery for their first 40 ticks (friction 0.05). That lets them slide off what they broke from instead of wedging against it.

Small pieces are also marked as debris. A piece is small when it holds `BlockGroupEntity.debrisMaxBlocks` blocks or fewer (3 by default). Debris that rests asleep and unheld for `debrisLifetimeTicks` (1200 ticks, one minute) crumbles away with particles and no drops. This keeps a demolished building from leaving thousands of single block groups behind.

Mined and burnt blocks do not make debris. Set `debrisMaxBlocks` to 0 to turn debris cleanup off, or call `setDebris` on a group to change its own flag.

## Carving

`carve(level, box)` removes every block whose centre lies inside a world box, as if mined but without drops, and returns how many went. Use it for scripted demolition, a tunnel bored through a falling mountain, or a cutscene. The usual splitting and structural checks follow.
