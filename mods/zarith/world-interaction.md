# World interaction

[Back to Zarith](../zarith.md)

This page covers how block groups deal with the world around them: entities, water and lava, fire, tools, explosions, arrows, weather and light. Everything here works for every `BlockGroupEntity`, and the hooks named along the way are listed on the [Block groups](block-groups.md#hooks) page.

## Entities

### Standing, walking and sliding

When a group is solid to entities (`wantsSolid()` returns `true`), every entity move in the game collides with its blocks as exact turned boxes. The collision itself comes from the [Solids API](solids-api.md), which handles a few things vanilla never had to:

* Surfaces up to 50° from flat can be walked up, like a ramp. Steeper ones stop you, and if you land on one you slide off it.
* Walking down a gentle slope follows it instead of turning into a string of small falls.
* Sneaking stops at the edge of a solid the way it does at the edge of a block.
* Standing on a solid does not count as flying for the server's anti cheat, and mining while standing on one is not slowed by vanilla's "not on the ground" penalty.

### Being carried

Entities standing on a moving group move with the point under their feet, including its rotation, so you can stand on a turning platform. Non player entities are carried on the server. The local player is carried on its own client, because vanilla lets each client move its own player; the server never moves players.

### Being pushed out

When a group moves into an entity, the entity is pushed out. I try several directions (up, the way the group moved, the face normals of each box the entity is inside, the four horizontal directions, then down) and take the shortest path that clears every box within 3 blocks without putting the entity inside terrain. A short lift up (0.6 blocks or less) counts as half its length, so entities get stepped up onto a group rather than shoved sideways.

An entity that was already more than 0.05 blocks inside a box before the group moved is left alone. It got there by its own doing, and pushing it out would launch it.

### Pushing a group

Players and mobs that walk into a solid group push it.

* A player's client checks each tick whether the player is walking into a group and, at most every 2 ticks, tells the server. The server checks the player is really touching it and applies `pushImpulse(1)` at the player's position, at most once per tick per player.
* Mobs push with half the impulse, scaled by how heavy they are compared to a player. An entity's mass is estimated from its bounding box: a player sized box is 70 kg, clamped to 5..4000 kg.

### Landing

An entity that lands on a group faster than 0.2 blocks per tick shoves it down where it landed, with an impulse of its mass times its landing speed (capped at 30 m/s) times `landingShare()`. The local player reports its own landings to the server.

### Weight on floating groups

On a group in water or lava, the weight of every entity standing on it pushes it down at that spot, scaled by `riderWeightShare()`. A raft tips when you walk to one edge. When the set of entities standing on it changes, a sleeping group wakes so it can settle again.

### Knockback

Explosions and anything else that changes an entity's vanilla motion change a group's motion too. The vanilla motion (blocks per tick) is multiplied by 12 and added to the body as velocity, capped at 25 m/s.

### Crushing

A group moving fast enough hurts living entities it comes down on. The check runs while the group is awake and its speed (linear speed plus spin times its size) is at least 4 m/s. For each nearby entity, only boxes above the entity's waist count, and the closing speed is the entity's own downward speed minus the box's. A closing speed under 4 m/s does nothing.

The damage comes from `crushDamage(closing)`: by default `2 + (closing - 4) × 1.5`, scaled by the group's mass from a quarter to double, at most 20. The damage type is `zarith:physics_block`. If a player held or threw the group in the last 200 ticks, the kill is credited to them ("was crushed by a block thrown by"). Turn crushing off with `crushes()`.

## Water and lava

Every 5 ticks while awake, or every 20 while asleep, a group checks whether any liquid is within half a block of it. Only then does it run the fluid code below.

### Sampling

Groups of 8 blocks or fewer sample each block at 8 points (a quarter block in from each corner). Bigger groups sample once per block, at its centre. Each sample measures how deep it is in fluid, from 0 to 1, using the fluid's real surface height.

### Forces

For every wet sample on a block that weighs:

* **Drag** pushes the sample toward the fluid's velocity: `fluidDrag(cell, lava) × depth × (flow × currentSpeed() + swell - velocity)`.
* **Buoyancy** lifts it: `buoyancy(cell) × gravity × depth`.
* **Heave damping** slows its vertical motion by 6 per second times its buoyant mass. Buoyancy alone is a spring with almost no damping, and without this a floating group would bob for a long time.

The forces act at each sample, so they also turn the group. A group partly in water rights itself, a long one lies along the current.

### Swell

When `swayAmount()` is above 0, still water carries two crossing swells. Their phase travels across the world, so different parts of a long group are on different parts of a wave and it rolls and turns, not just bobs. A group bobbing on the swell counts as resting for anything that waits for it to rest, and is never put to sleep by the slow motion rule.

### Splashes and lava

A block hitting water faster than 2 m/s downward makes a splash sound and particles, at most every 8 ticks. A block touching lava catches fire.

### Waking

A sleeping group in fluid wakes up when the fluid around it flows and it has a current speed, when the entities standing on it change, or when it is swaying.

## Fire

A group burns the way the world does, one block at a time.

### Where fire lives

Fire sits in open spots of the grid next to blocks, just as a vanilla fire block sits in an air block next to fuel. A group stores a map from grid spot to fire age (0 to 15). The client draws a vanilla fire block in each spot, turned so its "up" is the grid direction closest to the world's up.

### What lights it

* `ignite(cell)` or `ignite(cell, toward, again)` from code. The fire goes in the open spot next to the block that faces `toward`, or on top. `again` lights a block that already has fire next to it.
* `igniteNear(point, radius)` lights every block within the radius.
* Lightning within 2.5 blocks.
* Burning arrows, fireballs and other burning projectiles passing through a block.
* A fire block, lava or a lit campfire in the world at or next to a block. Checked every 10 ticks while awake, every 40 asleep.
* A burning group right next to it, one chance in three per check.
* Lava touching it.
* Flint and steel or a fire charge used on a block.

A block will not catch if it is not flammable (vanilla burn odds of 0), if it is wet (in water and not in lava), or if `burns()` returns `false`.

### How it spreads and burns

Each fire spot runs vanilla's `FireBlock.tick` logic inside the grid, with world directions mapped to grid directions through `gridFrame`. It only runs when the `doFireTick` game rule allows fire to spread there, and then each spot takes its turn about once every 35 ticks, matching vanilla's schedule.

* In rain, a spot near rain goes out with chance `0.2 + age × 0.03`.
* A spot with no flammable block next to it goes out once it is older than 3, or straight away if nothing is under it.
* Each flammable neighbour burns away with vanilla's odds (250 for blocks above or below, 300 for the sides, less in humid biomes). A burnt block is removed from the group with smoke, which may split the group.
* The fire spreads to open spots up to one block to each side and from one below to four above, with vanilla's chances.
* If `fireSpreadsToWorld()` is `true`, one in four spreading ticks also tries to place a real fire block in the world near the spot.

Living entities standing in a fire spot take fire damage and are set alight for 8 seconds, as they would in a fire block.

### Putting it out

* A fire spot that fills with water goes out with a fizz.
* Rain puts spots out over time, as above.
* A player can punch a fire spot out in one hit, as with a fire block.
* `burns()` returning `false` clears all fire.

On the client, burning groups give off smoke along the faces they burn on, rising campfire smoke and the fire crackle sound.

## Tools and clicks

### Right click

A right click on a group is handled in this order, and the first one that applies wins:

1. **Drag mode** (`/zarith drag`): pick the group up, or drop what you hold.
2. **An axe in either hand**: strips logs, wood, stems, hyphae and bamboo blocks, scrapes oxidised copper, or removes wax. The block changes in place and the group reshapes if its shape changed. `canStrip(cell)` can refuse it.
3. **Flint and steel or a fire charge in either hand**: lights the block you clicked.
4. **Link tool** (`/zarith link`): pick it for a joint.
5. **A vehicle seat**: gets in, if the group is a vehicle and the seat is free. Sneak right click never mounts.

### Mining

Holding left click on a group's block mines it like a world block. The client times it with vanilla's breaking speed for the held tool, draws the crack stages and plays the hit sounds. Instant break blocks break on the first click, and holding the button waits 5 ticks between blocks, as vanilla does.

When the client finishes, the block disappears at once on that client and the server is told. The server then checks that:

* the block is within the player's block reach plus 2.5 blocks (the group moved since the client aimed),
* the player is not mining faster than half the time the block should take.

If both pass, the block drops its items when the player has the right tool, the tool loses durability, the mined statistic and food exhaustion are applied, and the block is removed for everyone. If the server refuses, the block reappears on the miner's client after 30 ticks.

### Left click while holding

If you hold a group in drag mode, left click throws it. Otherwise, with drag mode on, left click shoves the group you are looking at.

## Explosions

There are two kinds of explosion and they act differently.

### Vanilla explosions

TNT, creepers, beds in the Nether and every other `ServerExplosion` reach groups through a mixin at the end of the explosion. Every group whose bounds are within twice the blast radius gets `explode(centre, radius, incendiary)`, which for each block within that range:

1. pushes the block away from the centre with an impulse of `mass × strength × 6`, where strength falls from 1 at the centre to 0 at twice the radius. The impulses are summed with the turning they cause, so a blast to one end spins the group;
2. breaks the block loose if `blastBreaks` says so: the blast's vanilla style damage at that distance against `4 + 3 × blast resistance`, with a random chance that grows with the excess;
3. on a load bearing group, wears the block down with `blastWear` if it did not break, so repeated blasts chew through a wall;
4. on a fiery explosion, sets one in three unbroken blocks within the radius alight.

Broken blocks drop items with a chance of `1 / radius`, like vanilla explosion drops, and are removed. The group splits wherever that leaves it in pieces. Groups ignore vanilla's own explosion knockback, since the impulses above already move them.

Groups and every other solid also shield what is behind them:

* When the blast casts its rays to choose which world blocks break, a point inside a solid counts that solid's blast resistance (or the world block's, whichever is higher).
* When the blast works out how much of an entity it can see, solids block the line of sight.

### /zarith explode

`/zarith explode` calls `PhysicsSpace.explode`, which only adds velocity to bodies. It breaks nothing, damages no one and leaves the world alone. It is a test tool.

## Arrows and projectiles

Arrows, tridents and other projectiles hit solids. The ray every projectile casts each tick also checks solids and returns whichever is closer, the solid or the world block.

An arrow that sticks into a group stays a real arrow entity: it can be picked up, it despawns, and it saves like any arrow. The group holds its position and heading in the grid frame and moves it along every tick. On the client the arrow is placed on the group's drawn pose every frame, so it does not trail behind.

* Picking it up, a loyal trident pulling itself out, or the arrow despawning ends the hold.
* When the group is destroyed, each arrow falls out moving at the speed of the spot it was stuck in.
* Arrows stuck in a group are saved with the group by UUID. After loading, the group waits up to 100 ticks for each arrow to load before forgetting it.
* An arrow stuck in a solid that is not a block group falls as soon as the solid moves away, because it sits in an air block that vanilla never checks again.

## Weather

* `isRainingAt` returns `false` under a solid whose top is more than half a block above the position. Fire, cauldrons and anything else that asks "is it raining here" see the shelter.
* Snow layers, ice and cauldron filling skip columns that a solid covers.
* On the client, rain and snow stop at the top of a solid. Vanilla's rain splashes under a solid are removed, and Zarith adds its own splashes on top of every solid instead.

## Light

On the client, groups that have been still for a second and are not being held darken the world under them:

* Sky light under a covered column drops as it would under real blocks: 14 at the edge of the covered area, one less for each column further in.
* World blocks next to a space a resting solid fills get ambient occlusion as if that space held a full block, so the ground darkens at the foot of a resting group.

The shade map is rebuilt every 5 ticks and only affected chunk sections are re-meshed. A tumbling group casts no shade at all, because every change would re-mesh chunks; `castsShade()` controls this.
