# Hover ships

[Back to Vehicles](vehicles.md)

A hover ship is a sci-fi drive. While powered it cancels gravity outright, so it floats wherever it is, and it pushes the body toward whatever velocity the pilot asks for. `HoverSettings` describes it and `PhysicsHover` runs it.

There is no sandbox command for hover ships yet, and I haven't tuned the defaults in game.

## Flying it

| Key | Effect |
|---|---|
| Jump (Space) | Rise. |
| V held | Sink. |
| W / S | Drive forward / back. |
| A / D | Turn. |
| Nothing | Brakes to a stop and holds still in the air. |

There is no throttle, no fuel and no stall. Movement is always level and straight up or down, whatever the ship is leaning, and the ship keeps itself upright. It leans into turns and dips its nose when driving forward, for looks only.

There is no strafing and no free rotation. The ship cannot fly upside down like a zero gravity spacecraft; that would need pitch and roll keys beyond the ones vehicles share.

Wheels on a hover ship are landing gear: always braked, no input.

## Parking

With `parkWhenEmpty` on (the default), the drive stays powered with nobody aboard, and the empty vehicle asks for no climb rather than the usual descent. Get out mid-air and the ship stays where you left it.

A ship left perfectly still may then fall asleep in Jolt while still in the air. That is intended: a sleeping body feels no gravity, so it stays put at no cost. Anything that bumps it wakes it, as does pressing a key or switching the drive off.

With `parkWhenEmpty` off, the drive powers down when the driver leaves and the ship falls. Switching the power off also wakes the body first, so a ship asleep in mid-air does not hang there unpowered.

## HoverSettings

Speeds and acceleration are the same whatever the ship weighs.

| Field | Default | Meaning |
|---|---|---|
| `topSpeed` | 25 | m/s forward. |
| `reverseSpeed` | 0.5 | Backward top speed, as a share of `topSpeed`. |
| `climbSpeed` | 8 | m/s up or down. |
| `acceleration` | 12 | m/s² the drive can change its velocity by, at most. |
| `grip` | 2 | How fast (1/s) the velocity follows what is asked. Low drifts like a ship on ice, high stops dead. |
| `yawRate` | 1.5 | rad/s of turn at full lock. |
| `turnInputRate` | 4 | Share of full lock per second the turn follows the keys. |
| `levelRate` | 4 | How fast (1/s) it rights itself. Low gives a looser, wallowing feel. |
| `bankAngle` | 0.3 | Radians banked into a full turn at speed. |
| `pitchAngle` | 0.1 | Radians nose down with forward held. |
| `attitudeResponse` | 8 | How fast (1/s) the body's spin follows what the drive asks. |
| `parkWhenEmpty` | `true` | Stay floating with nobody aboard. Saved as 1 or 0 in the settings array. |

## How it works

Each tick, while powered and awake:

1. **Weight cancelled.** A force of `mass × g` straight up against the space's gravity.
2. **Velocity.** The target is the level heading times `topSpeed` (or `-topSpeed × reverseSpeed` backward), plus up times `climb × climbSpeed`. The change toward it is `(target - velocity) × (1 - e^(-grip × dt))`, clamped to `acceleration × dt`, and applied as a force.
3. **Attitude.** The same as the helicopter's: the target up leans forward by `forward × pitchAngle` and banks by `turn × bankAngle`, the bank faded in until half of `topSpeed`. The spin toward it at `levelRate`, plus the turn at `yawRate`, is blended onto the body's spin at `attitudeResponse`. If the ship is on its nose or tail, it is first turned back toward level.

The heading is the ship's forward direction flattened onto the level, so pitching does not change where it drives.

## PhysicsHover

| Method | Notes |
|---|---|
| `control(climb, forward, right)` | All in [-1, 1]. Wakes the body when anything is pressed. |
| `setPower(boolean)` | Whether the drive runs. Turning it off wakes the body so it falls. |
| `step()` | Once a tick before the physics step. Does nothing while unpowered or asleep. |
| `powered()` | |
| `forwardSpeed()`, `climbRate()` | m/s, as of the last step. |
| `turn()` | As eased. |
| `body()`, `settings()`, `isDestroyed()` | |
