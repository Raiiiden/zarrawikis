# Saved data

[Back to Zarith](../zarith.md)

This page lists what a block group writes to NBT. You need it when you migrate saves, read groups with an external tool, or add fields of your own without breaking old worlds.

## Where the data sits

| Versions | Location in the entity's NBT |
|---|---|
| 1.20.1, 1.21.1 | Directly in the entity's own data, next to vanilla's `Pos`, `UUID` and so on. |
| 26.1.2, 26.2, 26.3 | Under one compound, `zarith` by default (`saveKey()`). The compound's contents are the same layout the older versions write. |

`PhysicsBlockEntity` is saved as entity `zarith:physics_block`.

## Group

| Key | Type | Contents |
|---|---|---|
| `Blocks` | compound | The grid, below. |
| `Origin` | list of 3 doubles | Where the grid's (0, 0, 0) corner is. |
| `Rotation` | list of 4 floats | Quaternion x, y, z, w. Normalised on load. |
| `RestingTicks` | int | Ticks spent resting. |
| `Age` | int | The entity's tick count. |
| `Asleep` | boolean | Whether the body was asleep. The new body starts asleep too. |
| `Velocity` | list of 3 floats | m/s. |
| `Spin` | list of 3 floats | rad/s. |
| `Fires` | long array | Fire spots, grid positions as `BlockPos.asLong`. Only when burning. |
| `FireAges` | int array | Age of each spot, same order. |
| `StuckArrows` | int array | 10 ints per arrow, below. Only when there are any. |
| `Joints` | list | Below. Only when there are any. |
| `Vehicle` | compound | Below. Only on vehicles. |
| `Flags` | byte | `PhysicsBlockEntity` only: 1 collision, 2 floats. |

Not saved: the physics body itself, block wear, a pending structural check, the debris flag and slip timer, and who threw the group.

## Grid (`Blocks`)

Only live blocks are written. Removed blocks are gone after a reload, so block ids are not stable across saves; only within one session.

| Key | Type | Contents |
|---|---|---|
| `Palette` | list of compounds | Block states, in the standard block state NBT form. |
| `Bricks` | int array | Three ints per brick that holds a live block: its brick x, y, z (grid position divided by 8). |
| `Masks` | long array | 24 longs per brick, in the same order: 8 for where its live blocks are, 8 for which are solid, 8 for which weigh. Each long is one 8 by 8 layer. |
| `States` | int array | A palette index per live block, walking bricks in order, then layers bottom to top, then bits low to high. |
| `Cut` | long array | Grid positions of removed solid blocks that can still break a diagonal bond. |

Inside a brick, a block's bit is `x + 8 × z` within the layer `y`, with x, y and z counted inside the brick (0 to 7). A block is `SOLID` if its solid bit is set, `SOFT` if only its mass bit is set, and `LOOSE` otherwise.

Groups saved before bricks existed have an int array `Cells` instead: five ints per block, x, y, z, palette index and `CellKind` ordinal. It is still read.

## Joints

Each element of `Joints` is a compound:

| Key | Type | Contents |
|---|---|---|
| `Other` | long array of 2 | The other group's UUID, most then least significant bits. Absent for a world pin. |
| `Kind` | string | `WELD`, `BALL`, `HINGE`, `ROPE` or `SPRING`. An unknown kind skips the joint. |
| `Data` | long array of 19 | Doubles as raw bits: anchor A x y z, anchor B x y z, axis A x y z, axis B x y z, min length, max length, frequency, damping, motor speed, motor torque, collide (1 or 0). |

A rope's `max length` is its measured length after its first build, so it keeps that length.

## Stuck arrows

Ten ints per arrow: its UUID as four ints, then its position in the grid frame (x, y, z) and its heading (x, y, z), as float bits.

## Vehicles

| Key | Type | Contents |
|---|---|---|
| `Settings` | int array | `VehicleSettings.toArray()` as float bits. |
| `Gears` | int array | Forward gear ratios as float bits. |
| `ReverseGears` | int array | Reverse gear ratios as float bits. |
| `Wheels2` | int array | 16 floats per wheel (`Wheel.toArray()`), as float bits. |
| `Seat` | long array of 3 | Seat x, y, z as double bits. |
| `WheelBlock` | string | Block id drawn for wheels. |
| `Ballast` | int array | kg, then the box's min x y z and max x y z, as float bits. Only with ballast. |
| `Boat`, `Helicopter`, `Plane`, `Rocket`, `Hover` | int array | That drive's settings array as float bits. Only when it has that drive. |
| `RocketFuel` | float | Fuel left, 0 to 1. Only on rockets. |
| `Gear` | int | Selected gear. Only with a manual gearbox. |

Floats are stored as raw bits on purpose: a wheel's unset values are NaN, and NaN must survive the trip.

A vehicle with no wheels and none of the five drives is dropped on load.

Vehicles saved before the settings became one array have `Wheels`, `Tuning`, `Tuning2` and `WheelSpecs` long arrays instead. They are still read. The old anti roll stiffness was absolute rather than per tonne, so it is not carried over and the new default applies.

## The settings array format

Every settings class has `toArray()` and `readArray(float[])` with the same rules:

* Element 0 is `ARRAY_VERSION`, currently 1 for every class. An array shorter than 2, or with a version below 1, is ignored and the defaults stay.
* The values follow in a fixed order.
* An array shorter than the current list keeps the defaults for whatever it lacks.
* A value that is NaN or infinite is skipped and keeps its default. (`VehicleSettings.centerSlipRatio` defaults to NaN for this reason: it can only be set to a finite value.)
* Booleans are stored as 1 or 0.

**To add a setting**, append it to the end of both the `toArray` list and the `readArray` list, and leave `ARRAY_VERSION` alone. Old saves then load with your default. Never reorder or remove a value: every save after it would shift. Bump `ARRAY_VERSION` only if you must change the meaning of existing positions, and handle the old version in `readArray`.

The same arrays go to clients in spawn data, so a server and client on different Zarith versions read each other's arrays by the same rules.

### Orders

`VehicleSettings`: version, kind ordinal, wheelRadius, wheelWidth, suspensionMin, suspensionMax, suspensionFrequency, suspensionDamping, maxSteerAngle, engineTorque, brakeTorque, handBrakeTorque, maxRpm, antiRoll, limitedSlipRatio, differentialRatio, manualGearbox, shiftTime, steerFalloffSpeed, steerRate, steerReturnRate, throttleRate, maxTilt, maxLean, caster, centerSlipRatio.

`VehicleSettings.Wheel` (always 16, no version): x, y, z, front, radius, width, suspensionMin, suspensionMax, suspensionFrequency, suspensionDamping, suspensionPreload, steer, drive, brakeTorque, handBrakeTorque, grip. A shorter array keeps only the position and `front`.

`BoatSettings`: version, propellerX, propellerY, propellerZ, rudderX, rudderY, rudderZ, keelX, keelY, keelZ, thrust, topSpeed, reverseShare, rudderForce, rudderSpeed, propWash, outboardAngle, keelDrag, yawDamping, throttleRate, rudderRate, rudderReturnRate, maxRpm.

`HelicopterSettings`: version, maxLift, climbSpeed, descendSpeed, climbResponse, maxTilt, bankAngle, topSpeed, levelRate, attitudeResponse, yawRate, spinUpRate, spinDownRate, tiltInputRate, turnInputRate, maxRpm.

`PlaneSettings`: version, thrust, topSpeed, stallSpeed, stallAngle, incidence, inducedDrag, maxLoad, sideDrag, controlSpeed, pitchRate, rollRate, yawRate, stability, rollLevel, attitudeResponse, throttleRate, stickRate, idleRpm, maxRpm.

`RocketSettings`: version, thrust, burnTime, terminalSpeed, steerRate, rcsRate, stability, finSpeed, upright, attitudeResponse, throttleRate, stickRate.

`HoverSettings`: version, topSpeed, reverseSpeed, climbSpeed, acceleration, grip, yawRate, turnInputRate, levelRate, bankAngle, pitchAngle, attitudeResponse, parkWhenEmpty.

## Older physics block saves

Physics blocks saved before blocks became groups hold a single block state and the entity's own position at the middle of the block's bottom face. On 1.20.1 and 1.21.1 the keys are `BlockState`, `Rotation`, `LinearVelocity` and `AngularVelocity`; on 26.x they are `block_state`, `flags`, `rotation`, `linear_velocity` and `angular_velocity`. `PhysicsBlockEntity` still reads them and turns the block into a one block group.
