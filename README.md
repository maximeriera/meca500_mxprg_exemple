# Meca500 `.mxprog` Examples

Annotated, ready-to-run example programs for the **Mecademic Meca500** — and the Meca500 only.

Every example is written to be **read as much as run**. The comments are the point: they explain *what* a command does, *why* you would choose it over the alternative, and *what you should see happen*. You should be able to open any file, read it top to bottom, and understand that feature without a robot in front of you.

Two audiences, one set of files:

- **Learning the robot?** Start at `01_basic/01` and read in order. By the end of `01_basic/` you can write your own programs; by the end of `03_advanced/` you can design a cell around one.
- **Demonstrating the robot?** Each file is self-contained and ends with concrete things to try, along with the question that feature answers.

> [!IMPORTANT]
> This is a collection of examples, not Mecademic product documentation.
> The authoritative reference is always the [Programming Manual](https://resources.mecademic.com/en/doc/MC-PM-MECA500/latest/).
> See [Status and licence](#status-and-licence).

## Contents

- [Meca500 `.mxprog` Examples](#meca500-mxprog-examples)
  - [Contents](#contents)
  - [Safety first](#safety-first)
  - [What you need](#what-you-need)
  - [What is an `.mxprog` file?](#what-is-an-mxprog-file)
    - [Comments](#comments)
    - [Where else these files come from](#where-else-these-files-come-from)
  - [What the command interface does *not* have](#what-the-command-interface-does-not-have)
  - [Conventions used in these examples](#conventions-used-in-these-examples)
  - [Folder structure](#folder-structure)
  - [The three setup programs](#the-three-setup-programs)
  - [Persistent vs. non-persistent settings](#persistent-vs-non-persistent-settings)
  - [File index](#file-index)
  - [Running an example](#running-an-example)
    - [Run it in simulation first](#run-it-in-simulation-first)
    - [If something goes wrong](#if-something-goes-wrong)
  - [Target hardware and firmware](#target-hardware-and-firmware)
  - [Status and licence](#status-and-licence)
  - [Official Mecademic resources](#official-mecademic-resources)

---

## Safety first

**These programs move a real robot.** Read this section before running anything.

- **Clear the workspace.** Every example opens by homing the robot. On a robot that has not been
  homed since power-on, that moves every joint through a large motion.
- **Run in simulation first** on any setup you have not tested. `ActivateSim(1)` executes
  everything with the motors doing nothing — see [Run it in simulation first](#run-it-in-simulation-first).
- **Check the poses against your cell.** The coordinates here assume a **bare flange and an
  empty workspace**. A tool, a fixture, or a part changes what is reachable and what collides.
  Nothing in this repo knows about your fixture.
- **Start slow.** Examples run at reduced speed on purpose. Raise the speed only once you have
  watched the path.
- **`00_common/00_cold_start.mxprog` can reset your safety envelope.** Its factory-reset section is
  commented out for exactly this reason. On a robot in a working cell, the work zone, joint
  limits and tool sphere may be deliberate protection — read the audit output before changing
  anything.

These examples are a learning aid. They are **not** a substitute for the risk assessment,
guarding and safety design your installation requires. You are responsible for the safety of
your own cell.

---

## What you need

- A **Meca500** (R3 or R4) on firmware **11.x**, ideally 11.3.
- A network connection to it. The robot's default address is `192.168.0.100`.
- A web browser, for the **MecaPortal** — the robot's built-in web interface. Nothing to
  install.
- That is all. There is no SDK, toolchain or licence needed to run these files.

Optional, and only for `03_intermediate/07`: an **MEGP 25E** or **MEGP 25LS** gripper.

---

## What is an `.mxprog` file?

An `.mxprog` file is a **plain-text program** saved from the MecaPortal code editor. It is a sequence of the robot's own text commands — the same commands documented in the [Programming Manual](https://resources.mecademic.com/en/doc/MC-PM-MECA500/latest/) — one per line.

There is no separate programming language and no compilation. What you type in the code editor is exactly what gets sent to the robot over TCP/IP.

```
// Move to the pick approach position
SetJointVel(25)
MoveJoints(0, -20, 30, 0, 40, 0)
```

Command names are **case-insensitive**, so `SetWrf`, `SetWRF` and `setwrf` all reach the same command. These examples use the manual's spelling throughout.

### Comments

Comments are **C/C++ style**, and they are most of what this repo is:

```
// A single-line comment

/* A block comment,
   used for the header of each example. */
```

`Ctrl` + `/` toggles comments on the selected lines in the MecaPortal editor.

### Where else these files come from

Not every `.mxprog` is hand-written. **RoboDK** exports to this format, and its output is recognisable: a generated header comment, whole-line comments carrying the original target names, and integer program names like `StartProgram(1)` rather than strings. It also emits comments where a movement could not be translated — for example, that linear movement using joint targets is not supported. Worth knowing if you meet a generated file and wonder why it looks nothing like these examples.

---

## What the command interface does *not* have

This is the single most important thing to understand before writing a program, and the most common surprise:

> The robot's command interface does **not** support conditionals, loops, or other flow control statements.

There is no `IF`, no `WHILE`, no `FOR`, no jumps or labels. An `.mxprog` program is a **linear sequence of commands**, executed in order.

That is a deliberate design choice. A Meca500 is a precision motion component intended to be driven by a PC, IPC or PLC that already owns the application logic:

- **Program logic lives in the host** — Python, C#, C/C++, Java, or Structured Text on a PLC. The host holds the state machine, the vision result, the recipe, the error handling.
- **The robot holds the motion** — the trajectories, frames, speeds and blending.
- `.mxprog` programs are the reusable motion building blocks the host calls into.

In practice this works well: the logic stays in a language you already have an IDE, a debugger, tests and version control for, and the motion stays in small named programs you can test one at a time. The trade-off is real, though — if you want the robot itself to branch on a sensor, it cannot, and the host has to.

The three tools that give you structure without flow control:

| Need | Mechanism |
|---|---|
| Reuse / modularity | `StartProgram("folder/program_name")` calls another saved program |
| Parameterised motion | Robot variables (beta): `MovePose(*vars.myGroup.pickPose)` |
| Application logic | The host program over TCP/IP, EtherCAT, EtherNet/IP or PROFINET |

`02_advanced/02_program_calls.mxprog` works through the architecture this implies.

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

Deactivating loses every non-persistent setting, so each example starts from firmware defaults regardless of what ran before it. That is the point: the file behaves the same on your bench as on a robot someone else used last. On an already-homed robot it does **not** move the arm — reactivation does not require re-homing.

`01_basic/01_first_program.mxprog` explains all five commands line by line and is the one to run first on a robot fresh from power-on. On such a robot it **homes for the first time, which moves every joint** — clear the area.

Two things this block does not do:

- It does not reset **persistent** settings — see [Persistent vs. non-persistent](#persistent-vs-non-persistent-settings). Use `00_common/00_cold_start.mxprog` for those.
- With an **MEGP 25\*** gripper fitted, every reactivation re-homes the gripper, so the fingers move on each run. The robot still does not.

`00_common/01_init.mxprog` deliberately has **no** reset block — it is called from inside another program's flow, and a deactivate there would tear down the state its caller just built.

<details>
<summary><b>To remove the reset block from every example</b></summary>

The block is byte-identical in all of them and always ends with `ResumeMotion()`. Delete from the `/* --- Standard reset` line through `ResumeMotion()` — one find-and-replace across the repo, or:

```bash
perl -0pi -e 's{/\* --- Standard reset.*?\nResumeMotion\(\)\n\n}{}s' */*.mxprog
```

That leaves `basic/01` untouched, which is correct — its copy is the teaching one, explained command by command, and is not the shared block.

</details>

**Header block** — every example opens with a block comment stating what it is for, what it needs, and what to watch:

```
/* ============================================================
   02 — MoveJoints: the joint-space move
   ------------------------------------------------------------
   ROBOT     Meca500 only (R3 or R4)
   TEACHES   What a joint move is and when to reach for it
   NEEDS     Robot activated + homed, no tool
   WATCH     All six joints start and stop together
   ============================================================ */
```

**Speeds** — examples run slow on purpose (typically 10–25%) so the motion is legible and safe
to stand next to. Every example says where to change the speed.

**Return to a known position** — examples end where they started, so you can run one after
another, or the same one twice, without re-jogging the robot.

**Reachability** — all poses sit well inside the Meca500 workspace and away from singularities.
The Meca500's mechanical joint limits are:

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
Start with `00_common/` for configuration knowledge.
Read `01_basic/` in order and you have the vocabulary.
Read `02_intermediate/` and you can build something useful.
Read `03_advanced/` and you can design a cell and answer an integrator's questions about it.

---

## The three setup programs

**`01_basic/01_first_program.mxprog`** takes the robot from powered-on to ready for motion: `DeactivateRobot()`, `ActivateRobot()`, `Home()`, `ResetError()`, `ResumeMotion()`. The last four are idempotent — none fails if the robot is already in the state it asks for.

**`00_common/01_init.mxprog`** puts the *settings* in a known state: frames, posture selection, speeds, blending, move mode, payload, torque limits. Save it on the robot as `00_common/01_init`,
then open your own programs with:

```
StartProgram("00_common/01_init")
```

**`00_common/00_cold_start.mxprog`** is for a robot you do not know — a loaner, a shared bench, anything someone else used last. It **audits** the persistent settings, then deactivates and reactivates to wipe every non-persistent one back to firmware defaults.

| | Scope | When |
|---|---|---|
| `01_basic/01_first_program` | Robot **state** — activate, home, clear fault | Once per power-up |
| `00_common/01_init` | **Non-persistent settings** — frames, speeds, blending | Start of every program |
| `00_common/00_cold_start` | Both, **plus** an audit of persistent settings | On an unfamiliar robot |

Settings are sticky, and they only reset **on activation**. A robot that stays activated all day never gets that reset, so between two runs the only thing that restores them is you — either explicitly (`init`) or by cycling activation (`cold_start`).

---

## Persistent vs. non-persistent settings

This is the distinction that decides what a reset can actually fix, and it is worth knowing before you debug a robot that is "behaving strangely".

**Non-persistent** settings live only as long as the robot is activated: TRF, WRF, speeds, blending, payload, torque limits, posture and turn selection, move mode. Deactivating loses all of them — which is what makes the standard reset block work, and why it does not move the arm.

**Persistent** settings are written to the robot's SD card and survive deactivation, reboots and power cycles. Only thirteen commands set them, and **deactivating does nothing to them** — they have to be dealt with deliberately. The ones that change how motion behaves:

| Command | Factory default | Why it bites |
|---|---|---|
| `SetJointLimits` / `SetJointLimitsCfg` | user limits **disabled** | a restricted joint changes every program, and can fail homing with `[1032]` |
| `SetWorkZoneLimits` / `SetWorkZoneCfg` | ±10,000 mm; `(4, 1)` | moves get refused for no visible reason |
| `SetToolSphere` | `(0,0,0,0)`, disabled | affects work-zone and collision checks |
| `SetCollisionCfg` | `4` | changes what a near-collision does |
| `SetCalibrationCfg` | `1`, enabled | disabled means Cartesian accuracy quietly misses the datasheet |
| `SetSimModeCfg` | `1`, normal | decides which mode `ActivateSim()` picks |

The table above covers eight of the thirteen. The remaining five are `SetRobotName`, `SetNetworkOptions`, `EnableEtherNetIp`, `EnableProfinet` and `SetOfflineProgramLoop` — the robot's identity, how it is wired into your network, and whether the button on its base loops program `1`. `cold_start` deliberately leaves all five alone: resetting them would take the robot off the network. (`SetOfflineProgramLoop` could not go there anyway — the robot only accepts it while a program is being saved with `StartSaving`.)

**Every one of the eight in the table can only be set while the robot is deactivated** — a running program must not be able to widen its own safety envelope. That restriction is also the opportunity: the deactivate step `cold_start` needs anyway is exactly the window in which they can be corrected.

(The other five vary. `SetRobotName` is also deactivated-only; `EnableEtherNetIp` and `EnableProfinet` run in any state; `SetNetworkOptions` is a queued command needing the robot ready for motion; and `SetOfflineProgramLoop` only works while a program is being saved with `StartSaving`. None of them affect motion, and `cold_start` touches none of them.)

> [!WARNING]
> `cold_start` leaves the factory-reset lines **commented out on purpose**.
> On a bare demo robot, uncomment them. On a robot in a working cell, several of them may be a safety envelope somebody
> configured deliberately — run the audit, look at the output, and decide before you change anything.

---

## File index

**`00_common/` — helpers**

| File | Purpose |
|---|---|
| `00_cold_start.mxprog` | Save as `00_common/00_cold_start`; persistent-settings audit plus a deactivate/reactivate reset |
| `01_init.mxprog` | Save as `00_common/01_init`; call with `StartProgram("00_common/01_init")` at the top of every program |

**`01_basic/` — the vocabulary**

| File | Teaches |
|---|---|
| `01_first_program.mxprog` | Readiness: `DeactivateRobot`, `ActivateRobot`, `Home`, `ResetError`, `ResumeMotion` — plus program anatomy and the motion queue |
| `02_move_joints.mxprog` | Joint-space motion — `MoveJoints` |
| `03_move_pose.mxprog` | Cartesian target, joint-space path — `MovePose` |
| `04_move_lin.mxprog` | Straight-line motion — `MoveLin`, and why it is *not* the default |
| `05_joint_motion_settings.mxprog` | `SetJointVel`, `SetJointAcc`, `SetJointVelLimit` — tuning in percentages |
| `06_linear_motion_settings.mxprog` | `SetCartLinVel`, `SetCartAngVel`, `SetCartAcc` — tuning in mm/s |
| `07_delay_and_sequencing.mxprog` | `Delay`, `SetCheckpoint`, and what "finished" actually means |

**`02_intermediate/` — making it useful**

| File | Teaches |
|---|---|
| `01_world_reference_frame.mxprog` | `SetWrf` — moving the origin, not the program |
| `02_tool_reference_frame.mxprog` | `SetTrf` — telling the robot where the tool tip is |
| `03_relative_moves.mxprog` | `MoveLinRelTrf` / `MoveLinRelWrf` — approach and retract |
| `04_posture_configuration.mxprog` | `SetConf`, `SetAutoConf` — eight ways to reach one pose |
| `05_turn_configuration.mxprog` | `SetConfTurn` — joint 6, and why cables fail |
| `06_blending.mxprog` | `SetBlending` — cycle time you get for free |
| `07_gripper.mxprog` | `GripperOpen`/`Close`, `MoveGripper`, force, and the parallel-execution trap |

**`03_advanced/` — into a real cell**

| File | Teaches |
|---|---|
| `01_variables.mxprog` | Robot variables (beta): `CreateVariable`, `vars.` and `*vars.` |
| `02_program_calls.mxprog` | `StartProgram` — composition, and how to architect a cell |
| `03_pick_and_place.mxprog` | A full cycle, assembled from everything above |
| `04_work_zone_and_collision.mxprog` | `SetWorkZoneLimits`, `SetToolSphere`, `SetCollisionCfg` |
| `05_payload_and_torque_limits.mxprog` | `SetPayload`, `SetTorqueLimits` — accuracy and process guarding |
| `06_time_based_motion.mxprog` | `SetMoveMode`, `SetMoveDuration` — moving on a fixed beat |

Two files are mostly prose, because the commands they cover cannot go in a running program: `03_advanced/04_work_zone_and_collision` (work zone and collision settings require the robot to be **deactivated**) and parts of `03_advanced/02_program_calls` (which needs sub-programs saved on the robot first). Both say so at the top and both still have a runnable section.

Every file ends with a **THINGS TO TRY** block — two or three concrete exercises with the robot, and the question that feature answers. They are the fastest way to make a concept stick, whether you are learning the robot or showing it to someone else.

---

## Running an example

1. Open the MecaPortal and connect to the robot **in control mode** — you cannot save or run programs from monitoring mode.
2. Load the `.mxprog` file into the code editor with the *load from computer* icon.
3. **Read the header block.** Check the preconditions match your setup, and that the poses are clear of anything you have mounted.
4. Press **run** — or select a few lines and press `Ctrl` + `Enter` to run only those, which is the best way to study one command at a time.

You do not need to activate or home first: every example does that itself.

To keep a program on the robot, save it. Program and folder names are case-sensitive, max 63 characters, from letters, digits and underscores — and a `/` in the name creates a folder, which is how `StartProgram("00_common/01_init")` finds its target.

### Run it in simulation first

Every example is written to be safe on a bare robot, but poses that are fine on one setup can collide with a fixture on another. Before running an unfamiliar program on real hardware, deactivate the robot and enable **simulation mode**:

```
ActivateSim(1)
```

The robot then executes everything — the 3D view moves, the log fills, errors and work-zone breaches are reported — with the motors doing nothing. `ActivateSim(2)` runs the same thing as fast as possible, which is the quickest way to check a long program for errors. `DeactivateSim()` returns to real motion. Both can only be issued while the robot is deactivated.

### If something goes wrong

The robot validates syntax **on save**, not as you type — a red dot on the tab and a red marker on the offending line means the robot rejected it. A motion error turns the run button red; click it (or *Reset error*) to send `ResetError`, then press resume to send `ResumeMotion`.

Common responses you will meet:

| Code | Meaning |
|---|---|
| `[1005]` | The robot is not activated |
| `[1006]` | The robot is not homed |
| `[1011]` | The robot is already in error |
| `[1032]` | Homing failed because joints are outside limits |
| `[3017]` | No offline program saved — `StartProgram` could not find its target |

---

## Target hardware and firmware

**These examples are for the Meca500 only.** Both the R3 and R4 are fine — the only difference that shows up here is that `SetJointVel` accepts up to 150 on an R4 and 100 on an R3.

Do not run them on any other Mecademic model. They are written throughout against the Meca500's six-axis geometry: six-value joint sets in every `MoveJoints`, the six mechanical joint limits listed above, three-parameter posture configuration (`SetConf`), and the wrist and shoulder singularities that go with that arm. None of that transfers to a different kinematic model, and a program that assumes it will either be refused or move somewhere you did not intend.

The firmware target is **11.3**. The examples are accurate across the 11.x family; the MecaPortal UI details and the variable commands are the parts most likely to have moved if you are on something older.

---

## Status and licence

**Licence: [MIT](LICENSE).** Copy these programs into your own projects, modify them, ship them. Attribution is appreciated but the licence only asks that you keep the copyright notice with substantial copies.

**Disclaimer.** This is a collection of examples. It is not Mecademic product documentation, it carries no warranty, and it is not a substitute for the official manuals or for Mecademic support. Where this repo and the Programming Manual disagree, the manual is right and this repo has a bug — please report it.

Note the warranty disclaimer in the licence is not boilerplate here: these files command a real robot, and what is safe on a bare arm on a bench may not be safe in your cell. Read [Safety first](#safety-first).

**Firmware drift.** These files were written against firmware 11.3. Commands, defaults and the MecaPortal UI change between firmware versions. If something here does not match your robot, check the manual for your firmware first.

**Found a mistake?** Open an issue. Corrections to the technical content are especially welcome — every claim here is meant to be traceable to the manual.

---

## Official Mecademic resources

Everything in this repo is derived from the documents below. They are the authority; this repo is a study aid. If the two disagree, believe the manual.

**Documentation**

| Resource | What it covers |
|---|---|
| [Programming Manual — MC-PM-MECA500](https://resources.mecademic.com/en/doc/MC-PM-MECA500/latest/) | Every command: syntax, arguments, defaults, usage restrictions. The reference these examples were checked against |
| [MecaPortal Operator Manual — MC-OM-MECA500](https://resources.mecademic.com/en/doc/MC-OM-MECA500/latest/) | The web interface: code editor, program manager, jogging, 3D view |
| [User Manual — MC-UM-MECA500](https://resources.mecademic.com/en/doc/MC-UM-MECA500/latest/) | Installation, mounting, wiring, specifications, and the safety chapter |
| [All technical resources](https://resources.mecademic.com/en/index.html) | Documentation hub, all models and firmware versions |

**Software**

| Resource | What it is |
|---|---|
| [Firmware downloads](https://resources.mecademic.com/en/firmware/index.html) | Firmware images and release notes |
| [mecademicpy](https://github.com/Mecademic/mecademicpy) | The official Python API — the natural next step once you need flow control |
| [Mecademic on GitHub](https://github.com/Mecademic) | Official drivers, APIs and integration examples |

**Support**

| Resource | What it is |
|---|---|
| [Knowledge base](https://support.mecademic.com/) | Articles, FAQs and troubleshooting |
| [mecademic.com](https://www.mecademic.com/) | Products, specifications, and how to contact Mecademic |

Documentation URLs above point at `latest`. To read the manual for a specific firmware version, replace `latest` with the version — for example `.../MC-PM-MECA500/11.1/`.
