# Helicopters

[Back to Vehicles](vehicles.md)

A helicopter is a rotor and tail rotor on a body. `HelicopterSettings` describes it and `PhysicsHelicopter` flies it.

The pilot does not work a collective and cyclic by hand. Instead they ask for a climb rate, a tilt and a turn rate, and the controller does what it takes to get them. The rotor lifts along the body's own up, so tilting the body is what carries it forward, as on a real helicopter.

There is no sandbox command for helicopters yet, and I haven't tuned the defaults in game.

## Flying it

| Key | Effect |
|---|---|
| Jump (Space) | Climb at up to `climbSpeed`. From the ground, this is also how you take off. |
| V held | Descend at up to `descendSpeed`. Held until it stops sinking, it lands. |
| W / S | Tilt nose down / up, which flies it forward / back. |
| A / D | Turn. Once moving, it also banks into the turn. |
| Nothing | Holds its height and levels out. |

### Taking off

On the ground the helicopter sits idle and its controller does nothing but spin the rotor up. The rotor spins up at `spinUpRate` while the engine runs. Asking to climb takes off once the rotor is fast enough for its lift to beat the weight, which is when `maxLift × rotor²` passes 1. With the defaults (`maxLift` 2, `spinUpRate` 0.25) that is after about 2.8 seconds in the seat.

### Landing

Hold V. When it has been asked to descend and has not been sinking faster than 0.25 m/s for 10 ticks, it counts as landed and stops flying. Its controller goes back to idle and the body rests on whatever is under it.

### Empty

The engine runs while a player sits in it, and an empty helicopter keeps it running for as long as it is airborne. With nobody aboard, the vehicle asks to descend, so an abandoned helicopter lets itself down and lands. After that the rotor winds down at `spinDownRate`.

## HelicopterSettings

Lift is a share of the body's weight, so a stone helicopter flies like a wooden one of the same shape.

| Field | Default | Meaning |
|---|---|---|
| `maxLift` | 2 | Full lift as a share of the weight. Above 1 is what lets it climb, and what it has spare to hold height while tilted. |
| `climbSpeed` | 5 | m/s climbing with jump held. |
| `descendSpeed` | 4 | m/s descending with V held. |
| `climbResponse` | 2 | How fast (1/s) the climb rate follows what is asked. |
| `maxTilt` | 0.35 | Radians nose down at full forward, or up at full back (about 20°). |
| `bankAngle` | 0.25 | Radians banked into a full turn. |
| `topSpeed` | 30 | m/s at which drag cancels full forward tilt: top speed in level flight. |
| `levelRate` | 3 | How fast (1/s) the body swings toward the attitude asked for. |
| `attitudeResponse` | 6 | How fast (1/s) the body's spin follows that. |
| `yawRate` | 1.2 | rad/s of turn at full pedal. |
| `spinUpRate` | 0.25 | Share of full rotor speed gained per second with the engine on. |
| `spinDownRate` | 0.1 | Share lost per second with it off. |
| `tiltInputRate` | 2 | Share of full tilt per second the tilt follows the keys. 0 follows at once. |
| `turnInputRate` | 3 | Same for the turn. |
| `maxRpm` | 400 | Rotor rpm at full speed. Dashboard and sounds only. |

## How it works

Once airborne, each tick:

1. **Lift.** The vertical acceleration wanted is gravity plus `(asked climb rate - current climb rate) × climbResponse`. The lift is that times the mass, divided by how upright the rotor is (never less than 0.3), so tilting does not make it sink. It is capped at `maxLift × weight × rotor²` and applied along the body's up.
2. **Drag on the level.** Horizontal motion is opposed by a force that matches the forward pull of full tilt exactly at `topSpeed`, growing with the square of speed. It never exceeds what would stop the motion in one step.
3. **Attitude.** The target up direction leans forward by `tilt × maxTilt` and sideways by the bank. The bank is `turn × bankAngle`, faded in from standstill to full at 30% of `topSpeed`, so it turns flat when hovering. The spin that rotates the body toward that target at `levelRate`, plus the turn at `yawRate`, is blended onto the body's spin at `attitudeResponse × rotor`. Spin is set directly rather than as torque, so the helicopter answers the same whatever its mass and shape.

While airborne with the rotor turning, the controller keeps the body awake, because a hovering body barely moves and Jolt would otherwise put it to sleep in mid-air.

Gravity comes from the space and the body's gravity factor, so a helicopter keeps working under `/zarith gravity` as long as gravity is not zero.

## Wheels

Wheels on a helicopter drive like a car's, from the same keys. Jump is then both the handbrake and climb. Skids made of blocks need no wheels at all.

## PhysicsHelicopter

| Method | Notes |
|---|---|
| `control(climb, forward, right)` | All in [-1, 1]. Held until changed. Wakes the body. |
| `setEngine(boolean)` | Whether the engine drives the rotor. |
| `step()` | Once a tick before the physics step. |
| `airborne()` | From takeoff until a held descent settles it. |
| `forwardSpeed()`, `climbRate()` | m/s, as of the last step. |
| `rotorSpeed()`, `rotorRpm()` | 0 to 1, and rpm. |
| `tilt()`, `turn()` | As eased. |
| `body()`, `settings()`, `isDestroyed()` | |
