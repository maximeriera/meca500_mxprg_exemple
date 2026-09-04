# Mecademic `.mxprog` Examples

Annotated, ready-to-run example programs for the **Meca500** — and the Meca500 only — written as training and demo material for the sales team.

Every example is written to be **read as much as run**. The comments are the product: they explain *what* a command does, *why* you would choose it over the alternative, and *what the customer should see happen*. A rep should be able to open any file, read it top to bottom, and be able to explain that feature on a call without a robot in front of them.

---

## What is an `.mxprog` file?

An `.mxprog` file is a **plain-text program** saved from the MecaPortal code editor. It is a sequence of the robot's own text commands — the same commands documented in the [Programming Manual](https://resources.mecademic.com/en/doc/MC-PM-MECA500/latest/) — one per line.

There is no separate programming language. What you type in the code editor is exactly what gets sent to the robot over TCP/IP.

```
// Move to the pick approach position
SetJointVel(25)
MoveJoints(0, -20, 30, 0, 40, 0)
```

### Where else these files come from

Not every `.mxprog` a rep sees will be hand-written. **RoboDK** exports to this format, and its output is recognisable: a generated header comment, whole-line comments carrying the original target names, and `StartProgram(1)` style integer program names rather than strings. It also emits comments where a movement could not be translated — for example, that linear movement using joint targets is not supported. Worth knowing, because a customer may arrive with a RoboDK-generated file and ask why it looks nothing like these examples.

### What the robot command interface does *not* have

This is the single most important thing to understand before writing examples, and the question a rep will get asked most often:

> The robot's command interface does **not** support conditionals, loops, or other flow
> control statements.

There is no `IF`, no `WHILE`, no `FOR`, no jumps or labels. An `.mxprog` program is a **linear sequence of commands**, executed in order.

That is a deliberate design choice, not a limitation to apologize for. Mecademic robots are components meant to be driven by a PC, IPC or PLC that already owns the application logic. The honest framing for a customer is:

- **Program logic lives in the host** — Python, C#, C/C++, Java, or Structured Text on a PLC.
  The host holds the state machine, the vision result, the recipe, the error handling.
- **The robot holds the motion** — the trajectories, frames, speeds and blending.
- `.mxprog` programs are the reusable motion building blocks the host calls into.

The three tools that give you structure without flow control:

| Need | Mechanism |
|---|---|
| Reuse / modularity | `StartProgram("folder/program_name")` calls another saved program |
| Parameterised motion | Robot variables (beta): `MovePose(*vars.myGroup.pickPose)` |
| Application logic | The host program over TCP/IP, EtherCAT, EtherNet/IP or PROFINET |

### Comments

Comments are **C/C++ style**, and they are the whole point of this repo:

```
// A single-line comment

/* A block comment,
   used for the header of each example. */
```

`Ctrl` + `/` toggles comments on the selected lines in the MecaPortal editor.

---

## Conventions used in these examples

**Units** — distances in millimetres, angles in degrees, time in seconds. Always. No inches, no radians, no configuration to get wrong.

**Every example resets the robot first.** All of them open with the same five-line block:

```
DeactivateRobot()
ActivateRobot()
Home()
ResetError()
ResumeMotion()
```

Deactivating loses every non-persistent setting, so each example starts from firmware defaults regardless of what ran before it. That is the point: the file behaves the same on your bench and on a robot someone else used last. It does **not** move the arm — a robot that was already homed needs no re-homing on reactivation.

`01_basic/01_first_program.mxprog` explains all five commands line by line and is the one to run first on a robot fresh from power-on. Note that on such a robot it **homes for the first time, which moves every joint** — clear the area before running it.

Two things this block does not do, both covered under [Persistent vs. non-persistent](#persistent-vs-non-persistent):

- It does not reset **persistent** settings. Use `00_common/cold_start.mxprog` for those.
- With an **MEGP 25\*** gripper fitted, every reactivation re-homes the gripper, so the fingers move on each run. The robot still does not.

`00_common/init.mxprog` deliberately has **no** reset block — it is called from inside another program's flow, and a deactivate there would tear down the state its caller just built.

**To remove the reset from every example**, the block is byte-identical in all of them and always ends with `ResumeMotion()`. Delete from the `/* --- Standard reset` line through `ResumeMotion()` — one find-and-replace across the repo, or:

```bash
perl -0pi -e 's{/\* --- Standard reset.*?\nResumeMotion\(\)\n\n}{}s' */*.mxprog
```

That leaves `01_basic/01` untouched, which is correct — its copy is the teaching one, explained command by command, and is not the shared block.

**Header block** — every example opens with a block comment stating what it teaches, what it needs, and what you should watch for:

```
/* ============================================================
   02 — MoveJoints: the joint-space move
   ------------------------------------------------------------
   TEACHES   What a joint move is and when to reach for it
   NEEDS     Robot activated + homed, no tool
   WATCH     All six joints start and stop together
   ============================================================ */
```

**Speeds** — examples run slow on purpose (typically 10–25%) so a demo is legible and safe to stand next to. Every example says where to change the speed.

**Return to a known position** — examples end where they started, so you can run one after another, or the same one twice, without re-jogging the robot.

**Reachability** — all poses are chosen well inside the Meca500 workspace and away from singularities, so an example does not fail on a customer's desk. The Meca500's mechanical joint limits are:

| Joint | Range |
|---|---|
| θ1 | −175° … 175° |
| θ2 | −70° … 90° |
| θ3 | −135° … 70° |
| θ4 | −170° … 170° |
| θ5 | −115° … 115° |
| θ6 | ±100 turns (no mechanical limit) |

---

## Folder structure

Examples are organised by **level**, and numbered so the reading order is the teaching order.

```
00_common/         Reusable helpers, not a level. init and cold_start.
01_basic/          What the robot does. One concept per file, nothing assumed.
02_intermediate/   How to make it useful. Frames, configurations, tooling.
03_advanced/       How it goes into a real cell. Composition, variables, safety.
```

Read `01_basic/` in order and you have the vocabulary for a discovery call. 
Read `02_intermediate/` and you can run a demo. 
Read `03_advanced/` and you can answer an integrator's questions.

### Three things to run before anything else

**`basic/01_first_program.mxprog`** takes the robot from powered-on to ready for motion:
`ActivateRobot()`, `Home()`, `ResetError()`, `ResumeMotion()`. All four are idempotent — none fails if the robot is already in the state it asks for — so the block is safe at the top of any program regardless of what state the robot was left in.

**`common/init.mxprog`** puts the *settings* in a known state: frames, posture selection, speeds, blending, move mode, payload, torque limits. Save it on the robot as `lib/init`, then open every program with:

```
StartProgram("lib/init")
```

**`common/cold_start.mxprog`** is the one to reach for on a robot you do not know — a loaner, a customer's bench, anything someone else used last. It **deactivates and reactivates**, which wipes every non-persistent setting back to firmware defaults in one stroke, then audits the persistent ones that deactivation cannot touch.

The distinction between these three is worth keeping straight:

| | Scope | When |
|---|---|---|
| `basic/01` | Robot **state** — activate, home, clear fault | Once per power-up |
| `common/init` | **Non-persistent settings** — frames, speeds, blending | Start of every program |
| `common/cold_start` | Both, **plus** an audit of persistent settings | On an unfamiliar robot |

Settings are sticky, and they only reset **on activation**. A robot that stays activated all day never gets that reset, so between two runs the only thing that restores them is you — either explicitly (`init`) or by cycling activation (`cold_start`).

<a name="persistent-vs-non-persistent"></a>
#### Persistent vs. non-persistent

This is the distinction that decides what a reset can actually fix.

**Non-persistent** settings live only as long as the robot is activated: TRF, WRF, speeds, blending, payload, torque limits, posture and turn selection, move mode. Deactivating loses all of them — which is what makes `cold_start` work, and it does **not** move the arm, because a robot that was already homed does not need homing again on reactivation. (With an MEGP 25\* gripper fitted, the gripper re-homes; the robot still does not move.)

**Persistent** settings are written to the robot's SD card and survive deactivation, reboots and power cycles. Only thirteen commands set them, and **deactivating does nothing to them** — they have to be dealt with deliberately. The ones that change how motion behaves:

| Command | Factory default | Why it bites |
|---|---|---|
| `SetJointLimits` / `SetJointLimitsCfg` | user limits **disabled** | a restricted joint changes every program, and can fail homing with `[1032]` |
| `SetWorkZoneLimits` / `SetWorkZoneCfg` | ±10,000 mm; `(4, 1)` | moves get refused for no visible reason |
| `SetToolSphere` | `(0,0,0,0)`, disabled | affects work-zone and collision checks |
| `SetCollisionCfg` | `4` | changes what a near-collision does |
| `SetCalibrationCfg` | `1`, enabled | disabled means Cartesian accuracy quietly misses the datasheet |
| `SetSimModeCfg` | `1`, normal | decides which mode `ActivateSim()` picks |

All thirteen can **only be set while the robot is deactivated** — a running program must not be able to widen its own safety envelope. That restriction is also the opportunity: the deactivate step `cold_start` needs anyway is exactly the window in which they can be corrected.

`cold_start` leaves the factory-reset lines **commented out on purpose**. On a demo robot, uncomment them. On a customer's production robot several of them are a safety envelope somebody configured deliberately — run the audit, show them the output, and let them decide.

### Contents

**`basic/` — the vocabulary**

| File | Teaches |
|---|---|
| `01_first_program.mxprog` | Readiness: `ActivateRobot`, `Home`, `ResetError`, `ResumeMotion` — plus program anatomy and the motion queue |
| `02_move_joints.mxprog` | Joint-space motion — `MoveJoints` |
| `03_move_pose.mxprog` | Cartesian target, joint-space path — `MovePose` |
| `04_move_lin.mxprog` | Straight-line motion — `MoveLin`, and why it is *not* the default |
| `05_joint_motion_settings.mxprog` | `SetJointVel`, `SetJointAcc`, `SetJointVelLimit` — tuning in percentages |
| `06_linear_motion_settings.mxprog` | `SetCartLinVel`, `SetCartAngVel`, `SetCartAcc` — tuning in mm/s |
| `07_delay_and_sequencing.mxprog` | `Delay`, `SetCheckpoint`, and what "finished" actually means |

**`intermediate/` — making it useful**

| File | Teaches |
|---|---|
| `01_world_reference_frame.mxprog` | `SetWrf` — moving the origin, not the program |
| `02_tool_reference_frame.mxprog` | `SetTrf` — telling the robot where the tool tip is |
| `03_relative_moves.mxprog` | `MoveLinRelTrf` / `MoveLinRelWrf` — approach and retract |
| `04_posture_configuration.mxprog` | `SetConf`, `SetAutoConf` — eight ways to reach one pose |
| `05_turn_configuration.mxprog` | `SetConfTurn` — joint 6, and why cables fail |
| `06_blending.mxprog` | `SetBlending` — cycle time you get for free |
| `07_gripper.mxprog` | `GripperOpen`/`Close`, `MoveGripper`, force, and the parallel-execution trap |

**`advanced/` — into a real cell**

| File | Teaches |
|---|---|
| `01_variables.mxprog` | Robot variables (beta): `CreateVariable`, `vars.` and `*vars.` |
| `02_program_calls.mxprog` | `StartProgram` — composition, and how to architect a cell |
| `03_pick_and_place.mxprog` | The full demo, assembled from everything above |
| `04_work_zone_and_collision.mxprog` | `SetWorkZoneLimits`, `SetToolSphere`, `SetCollisionCfg` |
| `05_payload_and_torque_limits.mxprog` | `SetPayload`, `SetTorqueLimits` — accuracy and process guarding |
| `06_time_based_motion.mxprog` | `SetMoveMode`, `SetMoveDuration` — moving on a fixed beat |

**`common/` — helpers**

| File | Purpose |
|---|---|
| `init.mxprog` | Save as `lib/init`; call with `StartProgram("lib/init")` at the top of every program |
| `cold_start.mxprog` | Save as `lib/cold_start`; deactivate/reactivate reset plus a persistent-settings audit |

Two files are mostly prose, because the commands they cover cannot go in a running program:
`advanced/04` (work zone and collision settings require the robot to be **deactivated**) and
parts of `advanced/02` (which needs sub-programs saved on the robot first). Both say so at
the top and both still have a runnable section.

Every file ends with a **TRY THIS IN FRONT OF A CUSTOMER** block: two or three concrete
things to do with the robot, and the question that feature is the answer to.

---

## Running an example

1. Open the MecaPortal and connect to the robot **in control mode** (you cannot save or run
   programs from monitoring mode).
2. **Activate** and **home** the robot.
3. Load the `.mxprog` file into the code editor with the *load from computer* icon.
4. Read the header block. Check the preconditions actually match your setup.
5. Press **run**, or select a few lines and press `Ctrl` + `Enter` to run only those — this
   is the best way to demo a single command.

To keep a program on the robot, save it: program and folder names are case-sensitive, max 63 characters, from `A–Z a–z 0–9 _ .` — and a `/` in the name creates a folder, which is how `StartProgram("demo/pick_and_place")` finds its target.

### Run it in simulation first

Every example here is written to be safe on a bare robot, but poses that are fine on one setup can collide with a fixture on another. Before running an unfamiliar program on real hardware, deactivate the robot and enable **simulation mode**:

```
ActivateSim(1)
```

The robot then executes everything — the 3D view moves, the log fills, errors and work-zone breaches are reported — with the motors doing nothing. `ActivateSim(2)` runs the same thing as fast as possible, which is the quickest way to check a long program for errors. `DeactivateSim()` returns to real motion. Both can only be issued while the robot is deactivated.

This is also a good demo in itself: a rep with no robot in the bag can still run the whole example set on a customer's own robot, or on a loaner, without touching anything.

### If something goes wrong

The robot validates syntax **on save**, not as you type — a red dot on the tab and a red marker on the offending line means the robot rejected it. A motion error turns the run button red; click it (or *Reset error*) to send `ResetError`, then press resume to send `ResumeMotion`.

---

## Target hardware and firmware

**These examples are for the Meca500 only.** Both the R3 and R4 are fine — the only difference that shows up here is that `SetJointVel` accepts up to 150 on an R4 and 100 on an R3.

Do not run them on any other Mecademic model. They are written throughout against the Meca500's six-axis geometry: six-value joint sets in every `MoveJoints`, the six mechanical joint limits listed above, three-parameter posture configuration (`SetConf`), and the wrist and shoulder singularities that go with that arm. None of that transfers to a different kinematic model, and a program that assumes it will either be refused or move somewhere you did not intend.

The firmware target is **11.3**. The examples are accurate across the 11.x family; the MecaPortal UI details and the variable commands are the parts most likely to have moved if you are on something older.

---

## Sources

- [Programming Manual — MC-PM-MECA500](https://resources.mecademic.com/en/doc/MC-PM-MECA500/latest/)
- [MecaPortal Operator Manual — MC-OM-MECA500](https://resources.mecademic.com/en/doc/MC-OM-MECA500/latest/)
- [Mecademic technical resources](https://resources.mecademic.com/en/index.html)
- [mecademicpy — the Python API](https://github.com/Mecademic/mecademicpy)
