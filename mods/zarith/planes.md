# Planes

[Back to Vehicles](vehicles.md)

A plane is an engine, wings and control surfaces on a body. `PlaneSettings` describes it and `PhysicsPlane` flies it. Thrust, lift and drag are real forces. The control surfaces and the tail's pull back into the airflow act on the spin, and fade with airspeed, so a plane too slow to fly cannot be steered and a stalled one drops its nose.

There is no sandbox command for planes yet, and I haven't tuned the defaults in game.

## Flying it

| Key | Effect |
|---|---|
| Jump (Space) held | Opens the throttle at `throttleRate` per second. |
| V held | Closes it. Held with the throttle already closed, it brakes the wheels. |
| W / S | Stick forward / back: nose down / up. |
| A / D | Roll left / right, with a little rudder in the same direction. |

The throttle stays where you leave it. The stick returns to centre when you let go. With the stick centred, the wings level themselves at `rollLevel`.

The engine only runs with a player in the seat. Getting out closes the throttle and the plane glides.

### Wheels and runways

A plane made with `new GroupVehicle(new PlaneSettings(), ...)` has no wheels and has to sit on its blocks. For a runway, start from a `VehicleSettings` with wheels and add the plane:

```java
VehicleSettings gear = new VehicleSettings();
gear.engineTorque = 0f;            // the propeller moves it, not the wheels
gear.wheel(4f, 0.5f, 5f, true);    // nose or front pair
gear.wheel(0f, 0.5f, 5f, true);
gear.wheel(4f, 0.5f, 1f, false);
gear.wheel(0f, 0.5f, 1f, false);
GroupVehicle plane = new GroupVehicle(gear, 2f, 1.5f, 3f).plane(new PlaneSettings());
```

On a plane the wheels never get throttle. They roll free, steer with A and D, and brake when the throttle is closed and V is held. Give them no engine torque.

## PlaneSettings

Forces are per tonne or a share of the weight, so a stone plane flies like a wooden one of the same shape.

| Field | Default | Meaning |
|---|---|---|
| `thrust` | 3500 | N per tonne at full throttle. |
| `topSpeed` | 40 | m/s where airframe drag balances full thrust in level flight. |
| `stallSpeed` | 15 | Airspeed at which the wings, at the stall angle, just carry the weight. Below it nothing keeps the plane up. |
| `stallAngle` | 0.26 | Angle of attack (radians, about 15°) of the most lift. Past it lift falls away. |
| `incidence` | 0.05 | Angle the wings are set at to the body, so level flight makes some lift. |
| `inducedDrag` | 0.1 | Drag from making lift, as a share of lift per unit of lift coefficient. |
| `maxLoad` | 4 | Most lift as a share of weight, so a hard pull cannot snap it round. |
| `sideDrag` | 2000 | N per m/s of sideslip per tonne: fuselage and fin resisting a skid. |
| `controlSpeed` | 12 | Airspeed at which the controls bite fully. Slower, they fade with its square. |
| `pitchRate` | 1.2 | rad/s at full stick. |
| `rollRate` | 2 | rad/s at full stick. |
| `yawRate` | 0.4 | rad/s of rudder that A and D add while rolling. |
| `stability` | 2.5 | How fast (1/s) the nose swings back into the airflow by itself: the tail's weathervane effect. |
| `rollLevel` | 0.5 | How fast the wings level with the stick centred. 0 for none. |
| `attitudeResponse` | 5 | How fast (1/s) the body's spin follows what the controls and tail ask. |
| `throttleRate` | 0.5 | Share of full throttle per second the keys move it. |
| `stickRate` | 4 | Share of full travel per second the stick follows the keys. 0 follows at once. |
| `idleRpm`, `maxRpm` | 800, 2600 | For the engine sound and gauges only. |

## How it works

Each tick, for the coming step:

1. **Thrust** along the nose: `thrust × tonnes × throttle`.
2. **Lift**, only with the air coming from ahead. The angle of attack is measured between the nose and the motion. The lift coefficient rises in a straight line from 0 to 1 at `stallAngle` (counting `incidence`), and past it falls off, but never below 0.3. Lift is `weight × (airspeed / stallSpeed)² × coefficient`, clamped to `maxLoad × weight`, and acts across the airflow over the wings.
3. **Drag** against the motion: the airframe's, `thrust × tonnes × (speed / topSpeed)²`, plus induced drag `|lift × coefficient| × inducedDrag`. Never more than stops it in a step.
4. **Side drag** against sideslip.
5. **Controls and stability**, faded by `(airspeed / controlSpeed)²` up to 1. Pitch and roll come from the stick, rudder from the roll input. The tail turns the nose toward the direction of motion at `stability`. With the stick centred, a roll toward level is added. The result is blended onto the body's spin at `attitudeResponse` times that fade.

A plane sitting still has no control at all. It needs speed before the stick does anything, which is why it needs a runway.

### Stalls

`stalled()` is true when the angle of attack plus incidence is past `stallAngle` and the airspeed is above 1 m/s. Past the stall the wings give less lift, and the tail keeps pulling the nose into the airflow, so a stalled plane drops its nose and picks up speed. Use `stalled()` and `angleOfAttack()` for a stall warning.

## PhysicsPlane

| Method | Notes |
|---|---|
| `control(throttle, pitch, right)` | Throttle up +1 / down -1 while held; stick forward +1 (nose down); roll right positive. Wakes the body. |
| `setEngine(boolean)` | Off closes the throttle. |
| `step()` | Once a tick before the physics step. |
| `airspeed()`, `climbRate()` | m/s, as of the last step. |
| `angleOfAttack()`, `stalled()` | Radians, nose above the airflow positive. |
| `throttle()`, `engineRpm()` | 0 to 1; rpm is 0 with the engine off. |
| `pitch()`, `roll()` | The stick as eased. |
| `body()`, `settings()`, `isDestroyed()` | |
