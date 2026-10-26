# Rockets

[Back to Vehicles](vehicles.md)

A rocket is an engine, a fuel tank, steering and fins on a body that stands on its tail. `RocketSettings` describes it and `PhysicsRocket` flies it.

There is no sandbox command for rockets yet, and I haven't tuned the defaults in game.

## Frame

A rocket's nose is the grid's **+Y**. Build it standing up. The engine pushes along +Y, and +Z is only the direction "stick forward" tips the nose toward. +X is still the left side.

## Flying it

| Key | Effect |
|---|---|
| Jump (Space) held | Opens the throttle at `throttleRate` per second. |
| V held | Closes it. |
| W / S | Tips the nose toward +Z / -Z. |
| A / D | Tips the nose left / right. |

The throttle stays where you leave it. With the stick centred, the rocket stands itself back up toward vertical, as fast as it can steer. The engine only runs with a player in the seat, and getting out closes the throttle. Out of fuel it coasts and falls, tumbling if nothing steers it.

Wheels on a rocket are landing legs: they are always braked and take no input.

## Fuel

The tank starts full. At full throttle it lasts `burnTime` seconds, and burns in proportion at less. Set `burnTime` to 0 for unlimited fuel; `fuel()` then always reports 1.

`GroupVehicle.fuel(share)` sets the fuel left, 0 to 1. It is saved with the group (`RocketFuel`) and carried over when the body is rebuilt. The dashboard shows it as a percentage.

## RocketSettings

Thrust is a share of the body's weight under normal gravity (9.81 m/s²), so a stone rocket flies like a wooden one of the same shape, and the engine still pushes where there is no gravity at all.

| Field | Default | Meaning |
|---|---|---|
| `thrust` | 2 | Thrust at full throttle as a share of the weight. Above 1 it lifts off. |
| `burnTime` | 60 | Seconds of fuel at full throttle. 0 never runs out. |
| `terminalSpeed` | 90 | m/s at which air drag equals the weight, so a falling rocket tops out near it. |
| `steerRate` | 0.8 | rad/s of tilt at full stick from the engine's gimbal at full throttle; in proportion at less, none with the engine off. |
| `rcsRate` | 0.3 | rad/s from steering thrusters that work with the engine off. 0 for none. |
| `stability` | 1.5 | How fast (1/s) the fins turn the nose into the airflow. 0 for no fins. |
| `finSpeed` | 40 | m/s at which fins reach full effect. Slower, they fade with the square of the speed. |
| `upright` | 0.5 | How fast (1/s) it stands back up with the stick centred, while it can steer. 0 for none. |
| `attitudeResponse` | 4 | How fast (1/s) the body's spin follows what steering and fins ask. |
| `throttleRate` | 0.5 | Share of full throttle per second the keys move it. |
| `stickRate` | 3 | Share of full travel per second the stick follows the keys. 0 follows at once. |

## How it works

Each tick:

1. The throttle moves with the keys and fuel drains by `throttle × dt / burnTime`.
2. While burning, the rocket is kept awake. A rocket sitting on the pad barely moves, and Jolt would otherwise put it to sleep in the middle of a burn.
3. **Thrust** along the nose: `thrust × mass × 9.81 × throttle`.
4. **Drag** against the motion: `weight × (speed / terminalSpeed)²`, never more than stops it in a step. Below 0.5 m/s there is none.
5. **Steering.** The steering rate is `steerRate × throttle + rcsRate`. The stick asks for that spin about the body's left (pitch) and forward (roll) axes. Spin about the nose itself is held at zero.
6. **Fins** add a turn toward the direction of motion, `stability × min(1, (speed / finSpeed)²)`.
7. **Upright.** With the stick centred and some steering available, a turn toward world up is added at `min(upright, steering rate)`.
8. The result is blended onto the body's spin at `attitudeResponse` times how much control it has (engine, thrusters and fins together, at most 1). With no gimbal, no thrusters and no airflow, nothing steers it.

## PhysicsRocket

| Method | Notes |
|---|---|
| `control(throttle, pitch, right)` | Throttle up +1 / down -1 while held; stick in [-1, 1]. Wakes the body. |
| `setEngine(boolean)` | Off closes the throttle. |
| `step()` | Once a tick before the physics step. |
| `burning()` | Engine on, throttle open and fuel left (or unlimited). For the flame and sound. |
| `speed()` | m/s along the nose. |
| `climbRate()` | m/s up, against gravity. |
| `throttle()` | 0 to 1. |
| `fuel()`, `setFuel(f)` | 0 to 1. |
| `pitch()`, `roll()` | The stick as eased. |
| `body()`, `settings()`, `isDestroyed()` | |
