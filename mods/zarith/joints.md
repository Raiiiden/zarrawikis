# Joints

[Back to Zarith](../zarith.md)

A joint holds two bodies together, or pins one body to a point in the world. Zarith builds them on Jolt's constraints. In the sandbox the [link tool](commands.md#link-tool) makes them by clicking; from code you make them through `JointSettings`.

## Kinds

| Kind | Holds | Jolt constraint |
|---|---|---|
| `WELD` | Both bodies in the pose they had when joined. | Fixed |
| `BALL` | The anchors together. The bodies swing freely about them. | Point |
| `HINGE` | The anchors together, turning about one axis only. Optionally driven by a motor. | Hinge |
| `ROPE` | The anchors no further apart than `maxLength`, and no closer than `minLength`. Slack in between. | Distance |
| `SPRING` | The anchors pulled back to `maxLength` by a spring. | Distance with min and max equal |

`BALL` is the default kind.

## JointSettings

`com.raiiiden.zarith.core.physics.JointSettings` is plain data, so you can save it and make the same joint again after a reload.

| Field | Default | Meaning |
|---|---|---|
| `kind` | `BALL` | |
| `anchorA` | (0, 0, 0) | Point on body A, in A's own frame. |
| `anchorB` | (0, 0, 0) | Point on body B, in B's own frame. **In world space when pinning to the world.** |
| `axisA`, `axisB` | (0, 1, 0) | Hinge axis in each body's frame (world space for a world pin). |
| `minLength` | 0 | `ROPE` only: how short it can get. |
| `maxLength` | -1 | `ROPE` length, or `SPRING` rest length. Negative means "however far apart the anchors are when the joint is made", which is filled in then. |
| `frequency` | 0 | `ROPE` and `SPRING` stiffness in Hz. 0 is rigid. |
| `damping` | 0.5 | 0 is bouncy, 1 does not overshoot. |
| `motorSpeed` | 0 | `HINGE` motor target, rad/s. |
| `motorTorque` | 0 | Torque the motor may use, N·m. 0 means no motor. |
| `collide` | `false` | Whether the two bodies still bump into each other. Usually not, because joined parts overlap at the joint. |

The builder methods return the settings for chaining:

```java
JointSettings hinge = JointSettings.of(JointSettings.Kind.HINGE)
        .anchors(0, 0.5, 0.5,  1, 0.5, 0.5)   // A's point, B's point
        .axes(0, 1, 0,  0, 1, 0)              // normalised for you
        .motor(4f, 5000f);                    // 4 rad/s, up to 5000 N·m

JointSettings rope = JointSettings.of(JointSettings.Kind.ROPE)
        .anchors(0.5, 0, 0.5,  0.5, 1, 0.5)
        .length(0f, 6f);                      // at most 6 m

JointSettings bungee = JointSettings.of(JointSettings.Kind.SPRING)
        .anchors(0.5, 0, 0.5,  0.5, 1, 0.5)
        .spring(1.5f, 0.2f);                  // rest length measured when made
```

`copy()` makes an independent copy.

### Frames

Anchors and axes are in each body's own frame, which for a block group is its grid. Block `(x, y, z)` fills the cube from `(x, y, z)` to `(x + 1, y + 1, z + 1)`, so the middle of the top face of block `(0, 0, 0)` is `(0.5, 1, 0.5)`.

A joint is built in world space from where the anchors are when it is made. So a weld keeps whatever pose the two bodies are in at that moment, and a hinge axis is fixed relative to each body from then on. If the two anchors are not at the same world point when a `BALL`, `HINGE` or `WELD` is made, Jolt pulls them together on the next step.

## Joints on block groups

On groups, use `link` rather than the space directly. The group then saves the joint and makes it again whenever both ends exist, so a rope survives either end unloading and loading back in.

```java
// Join two groups; anchorA is in a's grid, anchorB in b's grid
a.link(b, JointSettings.of(JointSettings.Kind.BALL).anchors(3, 1, 0.5, 0, 1, 0.5));

// Pin a group to the world; anchorB is a world position
a.link(null, JointSettings.of(JointSettings.Kind.BALL).anchors(0.5, 1, 0.5, x, y, z));

// Drop every joint on a, including ones other groups hold to it
int removed = a.unlinkAll();
```

The holding group keeps the joint as a `GroupJoints.Link` with the other group's UUID (or none for a world pin). Each server tick, any link without a live joint looks for its other end. If that group is not loaded yet, or cannot make a body yet, the link waits 20 ticks and tries again. When it succeeds the first time, the measured rope length is written back into the link's settings, so a rope keeps the same length forever after.

`joints().links()` lists a group's links; `Link.joint()` returns the live `PhysicsJoint` or `null` while the other end is missing.

Joints go away when:

* either body is destroyed (the core destroys them with it),
* the holding group is removed for good (killed or discarded, not unloaded); this also removes joints other groups hold to it,
* `unlinkAll()` is called.

Splitting is worth knowing about here. A joint belongs to one group and is anchored at a point in that group's grid. If the blocks around the anchor break off as a new piece, the joint stays with the original group, still anchored at the same grid point even though there are no blocks there now. The piece that broke off is free. Each rope link from `/zarith rope` is a group of its own, so mining one link removes it and its joints, and everything that hung from it falls.

## Joints in the core

Without groups, call the space:

```java
PhysicsJoint joint = space.createJoint(bodyA, bodyB, settings);   // bodyB null pins to the world
joint.destroy();
```

`createJoint` copies the settings, so later changes to your object do nothing. It returns `null` if either body is destroyed, belongs to another space, or both are the same body.

| `PhysicsJoint` method | Notes |
|---|---|
| `bodyA()`, `bodyB()` | `bodyB()` is `null` for a world pin. |
| `pinnedToWorld()` | |
| `other(body)` | The body at the other end. |
| `settings()` | A copy of what it was made from, with `maxLength` filled in. Enough to make it again. |
| `kind()` | |
| `worldAnchors(outA, outB)` | Both anchors in world space, for drawing a rope. |
| `setMotor(speed, torque)` | Changes a hinge's motor while it runs. 0 torque switches it off. Turning the motor on or off rebuilds the constraint; changing the speed alone does not. |
| `owner()`, `setOwner(Object)` | `GroupJoints` sets this to its `Link`. |
| `isDestroyed()`, `destroy()` | |

A joint is rebuilt automatically when either body changes shape, because the anchors are held relative to the old centre of mass.

## Collision between joined bodies

Bodies joined with `collide = false` do not collide with each other, but still collide with everything else, including other bodies in the same chain. Zarith gives every set of bodies linked this way a Jolt collision group in which only the pairs that are directly joined skip each other. So in a rope, each link ignores its two neighbours but still hits the link three along when the rope folds over itself. The groups are recomputed before the next step whenever such a joint is added or removed.

## What the sandbox builds with them

| Command | Joints used |
|---|---|
| `/zarith rope` | One `BALL` between each pair of links at their touching ends, and a `BALL` world pin at each end unless `loose`. |
| `/zarith cloth` | `BALL` joints at the middle of each shared edge between neighbouring tiles, slack `ROPE` joints across each diagonal so the sheet holds its shape, and two `BALL` world pins along the top edge of each top tile unless `loose`. |
| `/zarith link` | One joint of the chosen kind per pair of clicks. See [Commands](commands.md#link-tool). |
