# Boats

[Back to Vehicles](vehicles.md)

A boat is a propeller, a rudder and a keel acting on a body that floats. `BoatSettings` describes them and `PhysicsBoat` applies them. Jolt knows nothing about any of it.

There is no sandbox command for boats yet, and I haven't tuned the defaults in game. Expect to adjust them.

## Floating is the group's job

`PhysicsBoat` only drives and steers. It does not make anything float. Buoyancy comes from the group's `buoyancy(cell)` hook (see [Water and lava](world-interaction.md#water-and-lava)). With `PhysicsBlockEntity` that means passing the `FLOATS` flag:

```java
PhysicsBlockEntity hull = PhysicsBlockEntity.create(level, grid, origin, new Quaternionf(),
        PhysicsBlockEntity.COLLISION | PhysicsBlockEntity.FLOATS);
hull.setVehicle(new GroupVehicle(new BoatSettings()
        .propeller(1.5f, 0.2f, 0f)
        .rudder(1.5f, 0.2f, -0.3f)
        .keel(1.5f, 0.2f, 2.5f), 1.5f, 1f, 2f));
level.addFreshEntity(hull);
```

`FLOATS` settles every block 0.6 of its height under water whatever it is made of. A one block thick hull with its bottom at grid y = 0 has its waterline near y = 0.6, so the parts above sit at y = 0.2, under water. The numbers in this example are a sketch, not a tested boat.

## Each part works as far as it is under water

Every tick `GroupVehicle` measures how far under water the propeller, rudder and keel points are, from 0 to 1, over the half block around each point. Only water counts, not lava. Each part's force is scaled by that:

* a propeller lifted clear spins in the air and pushes nothing,
* a boat aground cannot steer,
* a keel out of the water stops resisting sideways slide.

## BoatSettings

Positions are in the grid frame (+Z forward, +Y up, +X left). Forces are per tonne of body mass, so a stone hull handles like a wooden one of the same shape.

| Field | Default | Meaning |
|---|---|---|
| `propellerX/Y/Z` | (0, -0.5, -2) | Where the thrust pushes from. Keep it low and aft, under the waterline. `propeller(x, y, z)` sets all three. |
| `rudderX/Y/Z` | (0, -0.5, -2.5) | Where the rudder pushes sideways. The further aft of the centre of mass, the harder it turns. `rudder(x, y, z)`. |
| `keelX/Y/Z` | (0, -0.5, 0) | Where the keel resists sliding sideways. Level with the centre of mass the boat turns flat; lower, it heels out of a turn as real hulls do, and a narrow one can capsize. `keel(x, y, z)`. |
| `thrust` | 3000 | N per tonne at full throttle from standstill. |
| `topSpeed` | 8 | m/s. Thrust falls to nothing at this speed. |
| `reverseShare` | 0.4 | Astern makes this share of the thrust and tops out at this share of the speed. |
| `rudderForce` | 1500 | Sideways N per tonne at full rudder with water passing at `rudderSpeed` or faster. |
| `rudderSpeed` | 4 | m/s of flow at which the rudder works fully. Slower, it fades with the flow. |
| `propWash` | 0.4 | Share of rudder force the propeller's own stream gives at full throttle. Lets a boat turn from rest. |
| `outboardAngle` | 0 | Radians an outboard motor swings its thrust with the rudder. 0 for a fixed propeller. |
| `keelDrag` | 3000 | N per m/s of sideways slide, per tonne. How firmly the hull carves instead of skidding. |
| `yawDamping` | 1.5 | Rate (1/s) at which spin about the up axis dies away with the keel fully wet. |
| `throttleRate` | 1.5 | Share of full throttle per second the throttle moves toward the helm. 0 follows at once. |
| `rudderRate` | 2.5 | Share of full rudder per second it moves out. |
| `rudderReturnRate` | 4 | Share per second it returns to centre. |
| `maxRpm` | 1200 | Propeller rpm at full throttle. For the dashboard and sounds only. |

## The forces

Each tick, for the coming step:

* **Thrust** along the hull, or swung by `rudder × outboardAngle` for an outboard: `thrust × tonnes × throttle × falloff × propellerWet`, where falloff runs from 1 at rest to 0 at top speed. Astern also multiplies by `reverseShare`.
* **Rudder** sideways at the rudder point. Turning right pushes the stern to the left (+X). Its strength follows the flow past it: `speed / rudderSpeed`, plus `propWash × throttle` when going ahead, clamped to ±1. Going astern reverses the flow, so the rudder works backwards as it does on a real boat.
* **Keel** against sideways slide at the keel point, never more than would stop the slide this step.
* **Yaw damping**, applied straight to the spin so a light hull cannot overshoot.

Forces at a point also turn the body, so a low keel heels the boat and an aft rudder swings the stern.

## PhysicsBoat

You only use this directly when driving a body without `GroupVehicle`.

| Method | Notes |
|---|---|
| `new PhysicsBoat(body, settings)` | |
| `control(accelerate, right)` | Helm: ahead +1, astern -1; right positive. Held until changed. Wakes the body. |
| `step(propellerWet, rudderWet, keelWet)` | Once a tick before the physics step, with each part's wetness 0 to 1. |
| `forwardSpeed()` | m/s along the hull, negative astern. |
| `throttle()`, `rudder()` | As eased, -1 to 1. |
| `propellerWet()`, `propellerRpm()` | For wakes, sounds and gauges. |
| `body()`, `settings()`, `isDestroyed()` | |

## Amphibious vehicles

Wheels and a boat on one vehicle both respond to the same keys. On land the wheels drive and the propeller is dry; in water the propeller pushes and the wheels have nothing to grip. The group still needs buoyancy to float.
