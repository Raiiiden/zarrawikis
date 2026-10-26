# Cars and motorcycles

[Back to Vehicles](vehicles.md)

Wheeled vehicles use Jolt's own vehicle constraint: cast wheels with suspension, an engine with a torque curve, a gearbox, differentials and tyre friction curves. The wheels are not bodies. Each step Jolt casts a cylinder down from each wheel's mount to find the ground, so wheels never snag on block edges.

`VehicleSettings` describes the vehicle. `PhysicsSpace.createVehicle(body, settings)` builds it on a body and returns a `PhysicsVehicle`. On a block group you do neither yourself: give the group a `GroupVehicle` and it builds the vehicle on its first tick.

## Laying out wheels

```java
VehicleSettings s = new VehicleSettings();
s.wheel(3f, 1.2f, 5.1f, true);    // front left   (x = 3 is the left side)
s.wheel(0f, 1.2f, 5.1f, true);    // front right
s.wheel(3f, 1.2f, 0.9f, false);   // rear left
s.wheel(0f, 1.2f, 0.9f, false);   // rear right
```

Each position is the wheel's **mount point** in the grid frame. The wheel hangs below it, its centre somewhere between `suspensionMin` and `suspensionMax` below the mount. `front` wheels steer by default and have no handbrake; rear wheels do not steer and do have one.

**For a car, add wheels in left and right pairs, front pair first.** Wheels 0 and 1 are a pair, 2 and 3 the next, and so on. Each pair shares a differential and, if `antiRoll` is above 0, an anti roll bar. With an odd count the last wheel has no partner.

**For a motorcycle, add wheels front to back down the middle.** Every driven wheel gets a differential of its own, and there are no anti roll bars.

`addWheel(x, y, z, front)` does the same as `wheel` but returns the `Wheel` so you can tune it:

```java
s.addWheel(0.5f, 1.2f, 3.3f, true)
        .drive(0f)                                           // free rolling front wheel
        .suspension(Float.NaN, Float.NaN, 1.5f, 0.5f);       // NaN keeps the vehicle value
```

## VehicleSettings

Torques are per tonne of body mass, and springs are sized from the weight each wheel really carries, so a stone car drives like a wooden one of the same shape.

| Field | Default | Meaning |
|---|---|---|
| `kind` | `CAR` | `CAR` or `MOTORCYCLE`. |
| `wheelRadius` | 0.5 | Metres. |
| `wheelWidth` | 0.4 | Metres. |
| `suspensionMin` | 0.05 | Shortest the spring gets: how close the wheel centre comes to the mount. |
| `suspensionMax` | 0.45 | Longest: how far below the mount the wheel hangs at full droop. |
| `suspensionFrequency` | 1.6 | How fast the body bobs on its springs, Hz. Higher is stiffer. |
| `suspensionDamping` | 0.5 | Damping ratio. 0 bounces, 1 does not overshoot. |
| `maxSteerAngle` | 0.55 | Front wheels' full lock, radians (about 31°). |
| `engineTorque` | 450 | Peak engine torque, N·m per tonne. |
| `brakeTorque` | 1500 | N·m per tonne, per wheel. |
| `handBrakeTorque` | 4000 | N·m per tonne, per rear wheel. |
| `maxRpm` | 6000 | Engine redline. |
| `antiRoll` | 0 | Anti roll bar stiffness per pair, N/m per tonne. 0 for none. Jolt applies the bars outside its solver and they bite hard: 50 all but stops a car rolling, and a few hundred can shake a vehicle apart. |
| `limitedSlipRatio` | 1.4 | How much faster one wheel of a pair may spin than the other. 1 locks them; large is an open differential. |
| `centerSlipRatio` | NaN | The same between pairs (a 4x4's centre differential). NaN uses `limitedSlipRatio`. |
| `differentialRatio` | 3.42 | Final drive. |
| `gearRatios` | 2.66, 1.78, 1.3, 1.0, 0.74 | Forward gears. |
| `reverseGearRatios` | -2.9 | Reverse gears. |
| `manualGearbox` | `false` | A manual gearbox stays in its gear until the driver shifts. |
| `shiftTime` | 0.25 | Seconds the clutch is out while changing gear. |
| `steerFalloffSpeed` | 0 | Above this speed (m/s) steering eases off, to half at twice the speed. 0 keeps full lock at any speed. |
| `steerRate` | 3 | Share of full lock per second the steering moves toward the keys. 0 follows at once. |
| `steerReturnRate` | 5 | Share of full lock per second it returns to centre. |
| `throttleRate` | 4 | Share of full throttle per second the throttle moves. |
| `maxTilt` | 60° | Radians the body may tip from upright before Jolt holds it there. `Math.PI` lets it roll over. |
| `maxLean` | 45° | Motorcycles: how far it may lean into a turn. |
| `caster` | 25° | Motorcycles: rake of the steering head back from vertical. |

### Wheel

Every `Wheel` field except the position defaults to NaN, which means "use the vehicle's value".

| Field / method | Default | Meaning |
|---|---|---|
| `radius(r)`, `width(w)` | vehicle | |
| `suspension(min, max, frequency, damping)` | vehicle | |
| `preload(metres)` | 0 | How far the spring is already compressed at rest. Lifts a heavy body. |
| `steer(radians)` | front: `maxSteerAngle`, rear: 0 | Full lock for this wheel. Negative steers against the front wheels (rear steering). |
| `drive(share)` | 1 | Its share of the engine relative to the other driven wheels. 0 for a free rolling wheel. Front, rear and all wheel drive are just different shares. |
| `brakes(brake, handBrake)` | vehicle; front wheels have no handbrake | N·m per tonne. |
| `grip(g)` | 1 | Tyre grip. 1 is a road tyre, below 1 slides more (ice), above 1 grips harder. |

## How Zarith sets up Jolt

Several of Jolt's defaults assume a 1.5 tonne car with 0.3 m wheels. Block vehicles can weigh anything, so Zarith scales them.

* **Springs from real load.** Jolt's frequency mode sizes a spring to the body's effective mass at the wheel, which on a wide or long body is far less than what the wheel actually carries, so the car sags onto its bump stops. Zarith works out each wheel's share of the weight at rest (the most even split that balances the body about its centre of mass) and sets stiffness and damping from that: stiffness `m × ω²`, damping `2 × ratio × m × ω`, with `ω = 2π × frequency`. A wheel far from the rest still carries at least 5% of an even share.
* **Torques per tonne.** Engine, brake, handbrake and anti roll stiffness are multiplied by the body's mass in tonnes (at least 0.05).
* **Inertia.** Wheel inertia scales with mass and with the square of the radius, so a heavy car's wheels do not overshoot each step and spin in place. Engine inertia and clutch strength scale with mass above 1.5 tonnes.
* **Shift points** for the automatic are at two thirds and one third of `maxRpm`.
* **Differentials.** A car gets one per pair that has a driven wheel; a motorcycle one per driven wheel. Each takes its share of the engine from its wheels' `drive` values, and within a pair the torque is split left and right by their `drive` values too.
* **Motorcycle fork.** With `caster` set, a steering wheel's suspension drops forward of its mount and it steers about that tilted axis.

## Driving logic

`PhysicsVehicle.control(accelerate, right, handBrake)` is the driver's keys. `GroupVehicle` calls it every tick.

With an **automatic** gearbox, `accelerate` is the throttle. Pressing against the way the vehicle rolls, above 1 m/s, brakes instead of driving the other way. Once nearly stopped, holding back reverses.

With a **manual** gearbox, the gear decides the direction. In a forward gear, forward is throttle and back is brake. In reverse it is the other way round. In neutral either key revs the engine. The clutch:

* is out while a shift is in progress (`shiftTime`) and in neutral,
* is in while there is throttle, or while rolling faster than 2 m/s (so lifting off gives engine braking),
* is out otherwise, so an idling engine does not creep the vehicle along.

`shiftUp()` and `shiftDown()` go one gear at a time, through neutral between first and reverse. `GroupVehicle` applies the driver's R and V presses this way and remembers the gear across saves.

Throttle and steering ease toward the keys at `throttleRate` and `steerRate`. Going from full left to full right passes through the centre at `steerReturnRate`. Easing and steering falloff happen in `refreshInput`, which the space calls just before each step; only values that changed since the last step are sent to Jolt.

### Motorcycle balance

Jolt has a lean controller for motorcycles, but it holds the bike up with a sideways push on the centre of mass. That push then does the cornering instead of the tyres, so the bike corners bolt upright. Zarith turns Jolt's lean controller off and balances the bike itself:

1. The target lean is the angle a real bike needs for the turn it is making: `atan(speed × yaw rate / gravity)`, clamped to `maxLean` and `maxTilt`, and smoothed so a jolt of yaw does not throw it about.
2. Each tick the roll rate is blended halfway toward closing the lean error in about 0.25 s, never faster than 2 rad/s.
3. While rolling, the bike turns about its tyres' contact line rather than its centre of mass, which means the centre of mass is moved sideways as it leans. Standing still, the tyres slide instead, so this is phased in up to 2 m/s.
4. Below 0.5 m/s and within 0.1 rad of the target, nothing is corrected, so a parked bike can fall asleep.

Balancing only runs while a wheel touches the ground. Jolt still limits steering to what the lean allows.

## PhysicsVehicle

| Method | Notes |
|---|---|
| `drive(forward, right, brake, handBrake)` | Raw input: forward and right in [-1, 1], brakes in [0, 1]. Held until changed. Wakes the body when forward or right is not 0. |
| `control(accelerate, right, handBrake)` | Keys, as described above. |
| `shiftUp()`, `shiftDown()`, `setGear(g)`, `gear()` | `gear()` is positive forward, negative reverse, 0 neutral. |
| `isManual()` | |
| `forwardSpeed()` | m/s along +Z, negative backing up. |
| `engineRpm()`, `maxRpm()` | |
| `throttle()`, `steering()` | As eased so far. |
| `wheelCount()` | |
| `steerAngle(i)` | Radians, positive to the left as Jolt measures it. |
| `rollAngle(i)`, `wheelSpeed(i)` | Rotation about the axle, and its rate. |
| `suspensionLength(i)`, `suspensionCompression(i)`, `suspensionForce(i)` | Compression is 0 fully extended, 1 fully in. Force in newtons this step. |
| `slip(i)` | 0 rolling, 1 locked or spinning in place. |
| `onGround(i)`, `contactPoint(i, out)` | |
| `setGrip(i, g)`, `grip(i)` | Changes a tyre's grip while running: a flat, a patch of ice. Kept in the settings. |
| `setTorqueScale(s)`, `torqueScale()` | Scales engine torque while running: a torque converter, a damaged engine. Survives a rebuild. |
| `settings()`, `body()`, `isDestroyed()`, `destroy()` | |

### Changing settings at run time

Jolt bakes most of the settings into the constraint when it is built: wheel positions, sizes, springs, brakes, gear ratios, differentials and anti roll bars. Changing those fields on a running vehicle has no effect until it is rebuilt. A rebuild happens when the body is reshaped (a block added or removed) or recreated, or when you destroy the vehicle and call `createVehicle` again.

These take effect without a rebuild: `setGrip`, `setTorqueScale`, the steering and throttle rates, `steerFalloffSpeed`, `manualGearbox` and `shiftTime`.

## The sandbox car and motorcycle

`Contraptions.car` and `Contraptions.motorcycle` (behind `/zarith car` and `/zarith motorcycle`) are worked examples of the settings.

**Car.** A 3 by 6 floor one block up, with a bonnet row on the front. Four wheels at the corners of the floor (half outside it, so the track is wide), radius 0.55, `suspensionMax` 0.6, front springs at 1.5 Hz and rear at 1.7 Hz so it dives a little under braking, `steerFalloffSpeed` 10. A quarter of its own weight again as ballast in a thin box under the floor between the axles. The seat is at (1.5, 2, 2.5).

**Motorcycle.** A one block wide frame three blocks long. Front wheel at z = 3.3, free rolling (`drive(0)`), 1.5 Hz; rear wheel at z = -0.3, driven, 1.8 Hz. Radius 0.55, width 0.2, `maxSteerAngle` 0.5, `steerFalloffSpeed` 6. Seat at (0.5, 2, 1.2).

Both are spawned five blocks ahead of the player, facing the way the player looks, with the floor's underside 0.3 below the spawn point so the wheels touch down as it settles, and with `PhysicsBlockEntity.COLLISION` so you can stand on them.
