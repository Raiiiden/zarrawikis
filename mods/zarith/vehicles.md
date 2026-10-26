# Vehicles

[Back to Zarith](../zarith.md)

Any block group can be made drivable by giving it a `com.raiiiden.zarith.group.GroupVehicle`. The vehicle holds a seat, settings for whatever moves it, and the controllers that turn the driver's keys into forces on the group's body.

There are seven kinds of drive, each with its own page:

| Drive | Settings class | Controller | Page |
|---|---|---|---|
| Car | `VehicleSettings` (`Kind.CAR`) | `PhysicsVehicle` (Jolt's wheeled vehicle) | [Cars and motorcycles](wheeled-vehicles.md) |
| Motorcycle | `VehicleSettings` (`Kind.MOTORCYCLE`) | `PhysicsVehicle` | [Cars and motorcycles](wheeled-vehicles.md) |
| Boat | `BoatSettings` | `PhysicsBoat` | [Boats](boats.md) |
| Helicopter | `HelicopterSettings` | `PhysicsHelicopter` | [Helicopters](helicopters.md) |
| Plane | `PlaneSettings` | `PhysicsPlane` | [Planes](planes.md) |
| Rocket | `RocketSettings` | `PhysicsRocket` | [Rockets](rockets.md) |
| Hover ship | `HoverSettings` | `PhysicsHover` | [Hover ships](hover-ships.md) |

Cars and motorcycles use Jolt's own vehicle simulation. The other five are controllers I wrote: Jolt knows nothing about them, and once a tick each one reads the body's state and adds forces and spin for the next step.

Cars and motorcycles have sandbox commands (`/zarith car`, `/zarith motorcycle`). Boats, helicopters, planes, rockets and hover ships are backend only for now: there is no command that spawns one, and I haven't tuned their default numbers in game yet. Treat the defaults as a starting point.

## Frame

Everything you give a vehicle is in the group's grid frame:

* +Z is forward,
* +Y is up,
* +X is the vehicle's **left**.

So a car facing +Z with its grid running from x = 0 to x = 3 has its left side at x = 3. The wheel positions, the seat, a boat's propeller and so on are all grid coordinates. A rocket is the exception for "forward": its nose points along +Y, and +Z is the way the stick tips it.

## Making a vehicle

```java
// A car or motorcycle: VehicleSettings with wheels
GroupVehicle car = new GroupVehicle(vehicleSettings, seatX, seatY, seatZ);

// The other kinds have their own constructors, with no wheels
GroupVehicle boat  = new GroupVehicle(new BoatSettings(), seatX, seatY, seatZ);
GroupVehicle heli  = new GroupVehicle(new HelicopterSettings(), seatX, seatY, seatZ);
GroupVehicle plane = new GroupVehicle(new PlaneSettings(), seatX, seatY, seatZ);
GroupVehicle rocket = new GroupVehicle(new RocketSettings(), seatX, seatY, seatZ);
GroupVehicle hover = new GroupVehicle(new HoverSettings(), seatX, seatY, seatZ);

// Or add a drive to a wheeled vehicle
GroupVehicle amphibious = new GroupVehicle(vehicleSettings, 1.5f, 2f, 2.5f).boat(new BoatSettings());
GroupVehicle aircraft = new GroupVehicle(wheels, 1.5f, 2f, 2.5f).plane(new PlaneSettings());

group.setVehicle(car);           // before level.addFreshEntity(group)
level.addFreshEntity(group);
```

Set the vehicle **before** spawning the group. The seat and all settings go to clients in the group's spawn data, so a vehicle set later is not known to clients that already see the group.

### Fields and methods

| Member | Notes |
|---|---|
| `settings` | The `VehicleSettings`. Always present, possibly with no wheels. |
| `boat`, `helicopter`, `plane`, `rocket`, `hover` | Public fields, `null` when the vehicle has no such drive. |
| `seat` | Where the driver sits, in the grid frame. The rider's feet go here. |
| `wheelState` | The block drawn for each wheel. Air (the default) draws nothing. |
| `ballast(kg, minX, minY, minZ, maxX, maxY, maxZ)` | Adds mass only weight filling a box in the grid frame. Use it to put the centre of mass low between the wheels. |
| `boat(...)`, `helicopter(...)`, `plane(...)`, `rocket(...)`, `hover(...)` | Add a drive. Return `this`. |
| `fuel(share)` | A rocket's fuel, 0 to 1. |
| `vehicle()`, `liveBoat()`, `liveHelicopter()`, `livePlane()`, `liveRocket()`, `liveHover()` | The running controllers on the server, or `null` before the first tick and on the client. |
| `wheelCount()` | |
| `wheelOnGround(i)`, `wheelCompression(i)`, `wheelSlip(i)` | Server side wheel readings, for skid marks and sounds. |
| `speed()`, `rpm()`, `gear()` | Client side: the dashboard as last sent. |
| `wheelPose(i, partialTick, centre, turn)` | Client side: a wheel's centre and turn for drawing. |

`BlockGroupEntity.setVehicle(null)` removes the vehicle and ejects the driver.

### When controllers are made

The settings objects are plain data. The controllers that use them are made on the group's first server tick, and made again whenever the group gets a new body (after loading, for example). Changing a field on a settings object changes the running vehicle from its next tick, with one exception: wheel layout and most `VehicleSettings` values are baked into Jolt's constraint when it is built. See [Cars and motorcycles](wheeled-vehicles.md#changing-settings-at-run-time).

## Driving

Right click a vehicle with your main hand to get in. Sneak right click never mounts, so you can still use tools on it. Sneak to get out, as with any vanilla vehicle. A vehicle has one seat and takes one passenger, the driver.

### Keys

| Key | Default | Cars and motorcycles | Boat | Helicopter | Plane | Rocket | Hover ship |
|---|---|---|---|---|---|---|---|
| Forward / Back | W / S | Throttle and brake | Ahead / astern | Tilt nose down / up | Stick forward / back | Tip nose forward / back | Drive forward / back |
| Left / Right | A / D | Steer | Rudder | Turn | Roll (with some rudder) | Tip nose left / right | Turn |
| Jump | Space | Handbrake | | Climb | Throttle up | Throttle up | Rise |
| Shift Down (Vehicle) | V | Gear down (manual) | | Descend | Throttle down; brakes wheels when closed | Throttle down | Sink |
| Shift Up (Vehicle) | R | Gear up (manual) | | | | | |
| Sneak | Shift | Get out | Get out | Get out | Get out | Get out | Get out |

R and V are Zarith's own key bindings, under Movement in the controls menu as "Shift Up (Vehicle)" and "Shift Down (Vehicle)". V does two jobs: each press is a gear change for a manual gearbox, and holding it is "down" for everything that flies or floats.

### How input reaches the server

1. Each client tick, `VehicleControls` reads the keys of the player in the driver's seat and packs them: accelerate (-1, 0, 1), right (-1, 0, 1), jump, lift (jump minus V held) and gear presses since the last send.
2. It sends them only when something changed, or when there are gear presses.
3. The server stores them in the vehicle (`GroupVehicle.input`). Values are clamped, and gear presses are capped at 8 either way.
4. Every server tick the vehicle hands the stored keys to each controller.

Until the first packet arrives, the server drives from the player's vanilla movement input instead, so a vehicle responds even before the client has sent anything. Only the player in the seat can drive; input from anyone else is ignored.

### What each controller is given

Every tick, `GroupVehicle` calls each controller that exists with `(lift, accelerate, right)` or the equivalent:

| Controller | Gets |
|---|---|
| Wheels on a plain car | `control(accelerate, right, handBrake)` |
| Wheels on a plane | `control(0, right, brake)`: they roll free and steer, and brake when the throttle is closed and V is still held. |
| Wheels on a rocket or hover ship | `control(0, 0, true)`: landing gear, always braked. |
| Wheels on a helicopter or boat | Treated as a car: they drive. |
| Boat | `control(accelerate, right)` |
| Helicopter | `control(lift, accelerate, right)` |
| Plane | `control(lift, accelerate, right)`: lift moves the throttle. |
| Rocket | `control(lift, accelerate, right)`: lift moves the throttle. |
| Hover ship | `control(lift, accelerate, right)` |

### With nobody in the seat

An empty vehicle gets `accelerate = 0`, `right = 0`, the handbrake on, and `lift = -1`. That parks a car, lets a helicopter set itself down, and closes a plane's or rocket's throttle. A hover ship with `parkWhenEmpty` gets `lift = 0` instead, so it stays where it was left.

Engines also depend on the driver:

| Drive | Runs when |
|---|---|
| Helicopter rotor | A player is driving, or it is still airborne (so an empty one keeps flying until it has landed). |
| Plane engine | A player is driving. |
| Rocket engine | A player is driving. |
| Hover drive | A player is driving, or `parkWhenEmpty` is on. |

### Combining drives

Wheels combine with any one other drive. Amphibious cars (wheels and a boat), planes with landing gear, rockets and hover ships standing on wheels all work.

Do not put more than one of helicopter, plane, rocket and hover ship on one vehicle. They share the jump and V keys, so every one of them would respond to the same keys at once. The dashboard only shows one of them.

## Dashboard

After the driver has sat for 3 seconds (so vanilla's "press Shift to dismount" message can be read first), the action bar shows a dashboard, refreshed every 2 ticks. The server sends the values with the wheel data each tick the vehicle is awake.

The first drive in this list that the vehicle has decides what is shown:

| Drive | Text (`en_us`) | Values |
|---|---|---|
| Helicopter | `120 km/h   Climb 3 m/s   Rotor 400 rpm` | Speed along the nose, climb rate, rotor rpm. |
| Plane | `150 km/h   Throttle 80%   Climb 5 m/s` | Airspeed, throttle, climb rate. |
| Rocket | `300 km/h   Throttle 100%   Fuel 42%` | Speed along the nose, throttle, fuel. |
| Hover ship | `90 km/h   Climb 2 m/s` | Forward speed, climb rate. |
| Wheels | `60 km/h   Gear D 3   4200 rpm` | Speed, gear (`N`, `R`, `R2`, or `D` and a number for an automatic), engine rpm. |
| Boat (no wheels) | `25 km/h   Propeller 900 rpm` | Speed, propeller rpm. |

The translation keys are `hud.zarith.helicopter`, `hud.zarith.plane`, `hud.zarith.rocket`, `hud.zarith.hover`, `hud.zarith.vehicle` and `hud.zarith.boat`.

The packet carries three numbers whatever the drive: a speed, an "rpm" and a "gear". For flying drives the "gear" slot carries the climb rate or the fuel, and the "rpm" slot carries a throttle percentage. Read `GroupVehicle.speed()`, `rpm()` and `gear()` on the client with that in mind.

## Saving and syncing

Every settings class can turn itself into a float array (`toArray`) and back (`readArray`). The arrays are what is saved in NBT and sent in spawn data. Older, shorter arrays keep the defaults for whatever they lack, so adding a setting does not break old saves. [Saved data](saved-data.md#vehicles) has the layout.

Besides settings, a vehicle saves a manual gearbox's gear and a rocket's fuel, so both survive a reload.
