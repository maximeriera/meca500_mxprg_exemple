# Disclaimer: what these examples simplify, and why

## Who these examples are written for

Someone with **little or no background in robotics or industrial automation** — a newcomer, a
student, an engineer from another discipline, someone evaluating a Meca500 for the first time.

The goal is that you can read a file top to bottom and come away able to *use* the feature and
*explain* it to somebody else. That goal shapes every explanation in this repo, and it
sometimes conflicts with being exhaustively precise.

**Where those two conflict, this repo chooses the explanation that leaves you with the right
instinct, over the one that is technically complete.**

That is a real trade-off and it deserves to be stated openly rather than discovered. This
document is the list of places where it happens.

## If you already know robotics

You will notice things here that are not quite right. Most of them are on the list below, and
they are deliberate. Reading this file first will save you the irritation.

The precise version of everything here is in the
[Programming Manual](https://resources.mecademic.com/en/doc/MC-PM-MECA500/latest/). It is well
written and unusually direct — if a simplification below matters to your application, go and
read the real thing. Nothing in this repo is a substitute for it.

---

## The simplifications

### Motion and paths

**"The joint with the furthest to travel sets the pace."**
The idea is right — joint moves are synchronised so every joint starts and stops together — but
the governing joint is the one that takes the **longest**, not the one that travels furthest.
How long a joint takes depends on its own velocity and acceleration limits as well as the
distance. A joint moving a short way slowly can govern a move over a joint sweeping further and
faster.

**"The tool tip travels an arc."**
Said of `MoveJoints` and `MovePose` throughout. The real path is a smooth curve determined by
the geometry, and it is generally **not** a circular arc. "Arc" is used because the thing that
matters to a beginner is *it is not a straight line*, and that is the intuition it buys.

**"`SetJointVel` is a percentage of maximum joint speed."**
It is a percentage of the **R3's top rated joint velocities**. That is why an R4 accepts values
up to 150 rather than 100 — the extra 50% is real additional speed on the newer robot, not an
overspeed. So `SetJointVel(100)` does not mean "as fast as this robot can go" on an R4.

**"Blending at 100% gives the roundest corner."**
Blending percentage is not a corner radius. 100% means blending occupies the whole of the
acceleration and deceleration phases. Because those phases depend on your speeds and
accelerations, **the same blending percentage produces a different-sized corner at different
speeds**. Treat the number as a dial to tune by eye, not a geometric specification.

**The motion queue as a simple FIFO.**
Commands are described as going into a queue and being executed one at a time in order. That
is the observable behaviour and the right mental model. Internally the controller looks ahead
across queued commands — that is how blending can start a corner before the previous move has
finished at all.

### Frames and configuration

**"The TCP is the tool tip."**
The TCP is the origin of the Tool Reference Frame, and it is wherever you define it with
`SetTrf`. Usually you put it at the physical tip, because that is the useful place. Nothing
requires it: for some applications it belongs at a pivot, a camera's focal point, or the centre
of a gripped part.

**"There are eight ways to reach a pose."**
Eight is the general case, and the number to remember. It is not universal: at or near a
singularity the count degenerates — there can be infinitely many joint sets for one pose — and
for poses near the edge of the workspace some of the eight are not reachable at all. The
examples say "usually eight" where it matters.

**Euler angles are treated as "three numbers you get from the MecaPortal".**
That is honest practical advice and it is what you will actually do. What is glossed over:
there are always **at least two, and sometimes infinitely many**, sets of Euler angles that
describe the same orientation. The robot accepts any of them but reports back a normalised set,
so **the numbers you read back may not be the numbers you sent**. In particular, when the second
angle is ±90° the robot always returns a first angle of 0. If you are comparing poses
numerically in host code, this will bite you, and the manual's section on Euler angles is
required reading.

**Singularities are named but never explained.**
The word appears throughout — "away from singularities", "`MoveLin` can be refused near a
singularity" — and is never defined, because a proper treatment needs the manipulator Jacobian
and would derail every file it appeared in. The working definition to carry around: *a robot
posture where the arm loses the ability to move in some direction, even though no joint is at
its limit*. Near one, small tool movements can demand enormous joint speeds. The
[Programming Manual](https://resources.mecademic.com/en/doc/MC-PM-MECA500/latest/) explains it
properly.

**Accuracy and repeatability are never distinguished.**
This is the single biggest omission in the repo, and it is the most common confusion among
people new to robots. Briefly:

- **Repeatability** — how closely the robot returns to the *same* point it went to before. This
  is the headline number on a datasheet, and it is very good on a Meca500.
- **Accuracy** — how closely the robot reaches the point you *asked for* in absolute
  coordinates. This is a different, larger, and harder number.

Most applications are repeatability-limited, which is why teaching points by jogging works so
well. Applications driven by a CAD model or a camera are accuracy-limited. If you are quoting a
number to anyone, know which one you are quoting.

### State and settings

**"Deactivating does not move the arm."**
Correct in the sense that matters: reactivating an already-homed robot does not execute a homing
motion. Deactivating does engage the brakes and de-energise the motors, which is a physical
event. Read the claim as *"no commanded motion"*, not *"perfectly rigid"*, and do not deactivate
with the arm somewhere that a millimetre would matter.

**Settings are described as simply "sticky".**
The full picture is the persistent/non-persistent split documented in the
[README](README.md#persistent-vs-non-persistent-settings). Individual example files mostly say
"this survives until something changes it" and leave the detail there.

**`SetJointAcc` as a plain percentage.**
There is a scaling factor inherited from firmware 8 that the manual documents (arguments need
multiplying by 1.43 if you are porting from firmware 7). Irrelevant unless you are migrating an
old program; omitted everywhere.

### Architecture

**The host/robot split is presented cleanly.**
"Logic in the host, motion in the robot" is the right architecture and genuinely why the robot
has no `IF`. What the examples do not dwell on is the **cost**: every decision the host makes
between two moves is a network round-trip, and a cycle that consults the host frequently will be
slower than one that does not. Designing to keep decisions out of the inner loop is real work.
The examples show the pattern; they do not show the tuning.

**"A `Delay` is a guess, a checkpoint is a fact."**
A fair summary, and the right instinct. To be exact: a checkpoint tells you the robot's motion
queue *reached that point*. That is what you want and it is genuinely reliable. Coordinating
with equipment outside the robot still needs the host to do the handshake — the checkpoint is
one half of it.

---

## What is *not* simplified

Some claims in this repo were deliberately kept precise even where a simpler line would have
read better. Do not treat these as approximations:

- **The work zone and collision features are not certified safety functions.** They prevent
  mistakes and protect equipment. They are not a substitute for a risk assessment, guarding, or
  the robot's actual safety signals. `advanced/04` says so, and it means it.
- **Torque limiting is a process guard, not a safety rating.** `advanced/05` says so.
- **Robot variables are a beta feature.** `advanced/01` says so, because someone designing a
  machine around them deserves to know.
- **Every command signature, argument range, default value and usage restriction** was checked
  against the Programming Manual for firmware 11.3. Where a file states a default or a limit,
  that is meant to be exact. If one is wrong, it is a bug — please report it.

---

## Found something wrong?

Two different things, and it helps to say which you think you have found:

- **A simplification** — something on the list above, or something that should be. If it misled
  you, that is worth knowing even though it was deliberate; the trade-off may have been called
  wrong.
- **An error** — a command that does not exist, an argument in the wrong order, a wrong default,
  a pose that cannot be reached, a claim the manual contradicts. These are bugs and there is no
  defence for them.

Open an issue either way. Corrections to the technical content are especially welcome.

Where this repo and the Programming Manual disagree, **the manual is right**.
