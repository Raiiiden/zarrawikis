# Commands

[Back to Zarith](../zarith.md)

Everything lives under `/zarith` and needs permission level 2 (operators, or cheats on in single player). The commands are a sandbox for testing physics and a set of worked examples for the API: each one is a few lines in `ZarithCommands` and `Contraptions`.

`/zarith` on its own does the same as `/zarith stats`.

## Flags

`spawn`, `physicify`, `assemble`, `rope` and `cloth` take flags at the end, in any order, separated by spaces:

| Flag | Also accepted | Effect |
|---|---|---|
| `collision` | `collide`, `solid` | Entities stand on, walk up and bump into the blocks. The blocks carry whatever stands on them, push entities out of their way, and get pushed when walked into or landed on. |
| `float` | `floats`, `floating` | The blocks float on water and lava, drift with currents and rock on the swell. Without it they still feel drag in fluids but sink. |
| `loose` | `free` | `rope` and `cloth` only: the ends are left free instead of pinned. |

Without `collision`, players and mobs walk straight through a physics block, though it still collides with terrain and other physics blocks.

## Making things

### /zarith spawn

```text
/zarith spawn <block> [pos] [count] [flags]
```

Spawns physics blocks of any block state, such as `minecraft:stone` or `minecraft:oak_stairs[facing=east]`. Without a position, it appears three blocks in front of you, or at the command's position for a command block or the console. With a count of 1 to 64 the blocks are stacked, each 1.02 blocks above the last.

### /zarith physicify

```text
/zarith physicify <from> <to> [flags]
```

Turns every block in the box into its own physics block, in place. It skips air, liquids, unbreakable blocks and anything with a block entity (chests, furnaces, signs, so no items are ever lost). At most 4,096 blocks are converted, and the box itself can hold at most 16,384 positions.

### /zarith assemble

```text
/zarith assemble <from> <to> [flags]
```

Turns every movable block in the box into **one** physics body: a house, a bridge, a statue. It takes the same blocks `physicify` does, up to 32,768 of them, from a box of at most 131,072 positions. Blocks with a collision shape become solid, the rest (torches, flowers, carpets) ride along. The body bears load, so mining or blowing out what holds part of it up makes that part fall. See [Destruction](destruction.md).

### /zarith rope

```text
/zarith rope <from> <to> [block] [flags]
```

Hangs a chain of physics blocks between two points, 0.5 to 64 blocks long, one link per block of length, ball jointed together, and pinned at both ends unless `loose`. The default block is the chain block (`iron_chain` on 26.x, `chain` before). Tall thin blocks run lengthwise; anything else lies flat along the rope, so `minecraft:oak_slab collision` makes a rope bridge you can walk across.

### /zarith cloth

```text
/zarith cloth [width] [height] [block] [flags]
```

Hangs a sheet of tiles, 1 to 16 each way (4 by 4 by default) in white carpet unless you give a block, from its top edge three blocks in front of you. Neighbouring tiles are ball jointed at the middle of their shared edge and tied across the diagonals with slack ropes, so it drapes, folds and flaps but holds together. The top edge is pinned unless `loose`.

### /zarith car and /zarith motorcycle

```text
/zarith car [block]
/zarith car manual [block]
/zarith motorcycle [block]
```

Spawn a drivable vehicle five blocks ahead, facing where you look, built of oak planks unless you give a block, with black concrete wheels. `manual` gives the car a manual gearbox: R and V change gear, and S is the brake until you shift into reverse. Right click to get in, WASD to drive, jump for the handbrake, sneak to get out. [Cars and motorcycles](wheeled-vehicles.md#the-sandbox-car-and-motorcycle) describes how they are built.

## Tools

### Drag mode

```text
/zarith drag [on|off]
```

Without an argument it toggles. While on:

| Action | Effect |
|---|---|
| Right click a physics block | Picks it up. It follows your crosshair at the distance you picked it up from, glowing, with its gravity off. |
| Right click anywhere while holding | Drops it. |
| Left click while holding | Throws it at 20 m/s where you look. |
| Sneak while holding | Reels it in, down to 1.5 blocks away. |
| Left click a block you are not holding | Shoves it at 6 m/s, slightly upward, with a random spin. |

You let go automatically if you walk or teleport more than 16 blocks from what you hold, die, or change dimension. Anything crushed by a block you held or threw in the last 10 seconds counts as your kill. Turning drag mode on turns the link tool off.

### Link tool

```text
/zarith link <weld|ball|hinge|motor|rope|spring|unlink|off>
/zarith link
```

`/zarith link` with no mode turns it off, like `off`. While a mode is on:

* Right click one physics object, then another, to join them. Rigid joints (weld, ball, hinge, motor) pivot on the point of the **second** click. Ropes and springs run between the two clicked points.
* Sneak right click an object to pin it to the world at the point you clicked.
* Right click the same object twice to cancel.
* Hinges turn about whichever world axis (X, Y or Z) is closest to the direction you are looking.

| Mode | Joint |
|---|---|
| `weld` | Holds the two in their current pose. |
| `ball` | Swings freely about the point. |
| `hinge` | Turns about one axis. |
| `motor` | A hinge driven at 4 rad/s with 2,500 N·m of torque per tonne of the two objects. |
| `rope` | Keeps the points no further apart than they were. |
| `spring` | Pulls the points back to their distance, at 1.5 Hz with 0.2 damping. |
| `unlink` | A click removes every joint on what you clicked. |

Joints made this way are saved with the objects. Turning the link tool on turns drag mode off. [Joints](joints.md) has the details.

## World and simulation

| Command | Effect |
|---|---|
| `/zarith explode [pos] [radius] [strength]` | Pushes physics objects away from a point: `strength` m/s at the centre (default 12) falling to nothing at `radius` (default 6, up to 64). Breaks nothing and hurts no one. |
| `/zarith ignite [radius]` | Sets physics blocks within the radius (default 3) of you alight. |
| `/zarith gravity <x> <y> <z>` | Sets gravity for this dimension's physics, in m/s², each axis -100 to 100. Only affects physics bodies, not entities. Not saved: it resets when the server restarts. |
| `/zarith gravity reset` | Back to (0, -9.81, 0). |
| `/zarith pause` / `/zarith resume` | Freezes or resumes physics in this dimension. Not saved. |
| `/zarith clear [radius]` | Removes `zarith:physics_block` entities, all of them in this dimension or those within the radius of you. Other mods' group types are left alone. |
| `/zarith stats` | Jolt's version, and for this dimension: bodies, awake bodies, joints, vehicles, terrain colliders, the last step's time in milliseconds, gravity, and whether it is paused. |

If Jolt could not load, `stats` and the commands that need physics say why instead.
