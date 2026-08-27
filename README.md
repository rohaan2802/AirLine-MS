# AirLine Managment System (AirControlX)

Operating Systems project: an **airport / ATC simulation** in one translation unit. Flights move through phases, three **runways** each run a **worker thread** with a **mutex**, ATC uses **priority queues**, and an **SFML** window (600×500, title **AirControlX Simulation**) draws runway occupancy. Speed-limit breaches set a violation flag; an **airline portal** can list/pay **AVN** records if any were generated.

**Main source:** `OS PROJECT_AIR LINE MANAGEMENT SYSTEM.cpp`  
**Spec:** `OS_Project_Statement -SP2025.pdf` (~4.1 MB on GitHub; behaviour below is from the `.cpp`, including comments tagged `FR2.1` / `FR2.3`)  
**Author:** Mohammad Rohaan (22I-2327) · [rohaan2802](https://github.com/rohaan2802)

---

## Table of contents

1. [What this program does](#what-this-program-does)
2. [Architecture](#architecture)
3. [Enums, constants, airlines](#enums-constants-airlines)
4. [`Flight`](#flight)
5. [`Runway` (threads and sync)](#runway-threads-and-sync)
6. [`AVN` and `AirTrafficControl`](#avn-and-airtrafficcontrol)
7. [`Simulation`](#simulation)
8. [Interactive `main` / `createFlight`](#interactive-main--createflight)
9. [SFML visualization](#sfml-visualization)
10. [File-by-file](#file-by-file)
11. [Build and run](#build-and-run)
12. [Limitations](#limitations)
13. [Author](#author)

---

## What this program does

1. `srand(time(0))`; open SFML window; construct `Runway` **A/B/C** and `AirTrafficControl`.
2. Optionally loop `createFlight` (airline, direction, `HH:MM`).
3. **Require at least one cargo** flight (`hasCargoFlight`) or `return 1`.
4. `Simulation::start()` for **`SIMULATION_DURATION` (300 s)** or until the window closes: 1 s ticks, fuel/speed updates, runway assignment, optional ground fault, console dumps every 5 s, SFML redraw.
5. Print a final schedule; `airlinePortal` (pay AVN by id).
6. `delete` each `Flight*`.

This is a **threaded simulation**, not a real airline reservation UI. There is no separate process-per-flight (`fork`); concurrency is `std::thread` + `mutex` + `atomic`.

---

## Architecture

```text
main
  Runway rwyA, rwyB, rwyC          each starts runwayThreadLoop
  AirTrafficControl controller
  Simulation simulation(&controller, &window)
  createFlight → pending[] → start()
        scheduleFlights / assignRunway
        Flight::updateFuel / progressPhase
        renderVisualization (SFML)
  airlinePortal(payAVN)
```

Headers used: `<thread>`, `<mutex>`, `<atomic>`, `<chrono>`, `<queue>`, `<SFML/Graphics.hpp>`.

Helpers: `formatTime` (`localtime_r` + `put_time` `%H:%M:%S`), `getDirectionMeaning`, `getAircraftTypeName`.

---

## Enums, constants, airlines

```text
AircraftType   COMMERCIAL, CARGO, EMERGENCY
FlightPhase    HOLDING, APPROACH, LANDING, TAXI, AT_GATE, TAKEOFF_ROLL, CLIMB, CRUISE
Direction      NORTH, SOUTH, EAST, WEST, ARR, DEP
RunwayID       RWY_A, RWY_B, RWY_C
FlightStatus   WAITING, ACTIVE, COMPLETED
```

| Constant | Value | In source |
|----------|-------|-----------|
| `NorthTime` | 180 | Declared (“every 3 min”); **not read** by `Simulation::start` |
| `SouthTime` | 120 | Same |
| `EastTime` | 150 | Same |
| `WestTime` | 240 | Same |
| `SIMULATION_DURATION` | 300 | Loop bound (seconds) |
| `FUEL_THRESHOLD` | 100.0f | Promote to emergency |

Spawn interval constants look like a spec timetable; **actual flights are only those entered in `createFlight`**. There is no automatic NORTH-every-180s generator.

Seeded `vector<Airline> airlines`:

| Name | Type | Aircraft | Active flights field |
|------|------|----------|----------------------|
| PIA | COMMERCIAL | 6 | 4 |
| AirBlue | COMMERCIAL | 4 | 4 |
| FedEx | CARGO | 3 | 2 |
| PAF | EMERGENCY | 2 | 1 |
| Blue Dart | CARGO | 2 | 2 |
| AghaKhan Air | EMERGENCY | 2 | 1 |

`Airline` also tracks `avnCount`.

---

## `Flight`

Constructor: arrivals get random speed `400 + rand()%201`; departures speed 0. Phase **HOLDING** if arrival (`NORTH`/`SOUTH`/`ARR`), else **AT_GATE**. Fuel: commercial **12000**, cargo **15000**, emergency **10000**.

| Method | Role |
|--------|------|
| `calculateFuelConsumption` | Per-phase burn (holding 1.2 … takeoff/climb 2.0) |
| `generateRandomSpeed` | Phase bands; **20%** chance of an out-of-band “violation” sample when `allowViolation` |
| `isArrival` / `isDeparture` | Direction checks |
| `updateFuel` | Burn × elapsed seconds; new speed; if fuel &lt; threshold and not already emergency → type `EMERGENCY`, direction `ARR` or `DEP` |
| `updateSpeed` / `checkViolation` | Compare speed to phase limits; first violation sets `hasViolation` and `airline->avnCount++` |
| `progressPhase` | Arrival: Holding→Approach→Landing→Taxi→Gate→COMPLETED. Departure: Gate→Taxi→Takeoff→Climb→Cruise (COMPLETED) |
| `getPhaseDuration` | 10 or 8 seconds per phase (cruise 0) |
| `getPriority` | Emergency 0, cargo 1, commercial 2 |
| `assignRunway` / `setStatus` | Setters |

Phase speed limits used in `checkViolation` (km/h): Holding 400–600, Approach 240–290, Landing 30–240, Taxi ≤30, Gate ≤10, Takeoff ≤290, Climb ≤463, Cruise 800–900.

`checkViolation` does **not** call `AirTrafficControl::generateAVN`. The portal’s AVN list stays empty unless something else pushes into `avns`.

---

## `Runway` (threads and sync)

Each runway:

- `mutex runwayMutex`
- `atomic<bool> occupied`
- `Flight* currentFlight`
- `thread runwayThread` + `atomic<bool> running`

Ctor sets `running = true` and starts `runwayThreadLoop`: every 1 s, if occupied ≥ **10 s**, force-release (log `[Runway id] Forced release after 10s`). Dtor sets `running = false` and `join()`.

`tryOccupy`: lock; if free, occupy; else if already occupied ≥ 10 s, steal the strip. `releaseRunway` clears occupant.

This is the project’s **synchronization** story: mutex around occupancy, atomic flags, a dedicated thread per runway, plus `AirTrafficControl::queueMutex` around queues.

---

## `AVN` and `AirTrafficControl`

**`AVN`:** id, flight number, airline, type, recorded vs permissible speed, `issuanceTime`, `fineAmount`, `paymentStatus` (`"unpaid"`), `dueDate` = now + **3 days**. Fine: `(COMMERCIAL ? 500000 : 700000) * 1.15` (15% service fee).

**`AirTrafficControl`:** pointers to three runways; `vector<AVN> avns`; nested `PriorityQueue` with

- `priority_queue` of emergencies (`EmergencyComparator` = later `scheduledTime` has lower priority — earliest sched first),
- `queue` cargo,
- `queue` commercial.

`addFlight` pushes onto **arrival** or **departure** queues by type.

`processQueue`: drain emergencies while `assignRunway` succeeds; then one pass over cargo; then commercial. (The commercial failure path calls `queue.cargo.pop/push` — likely a copy-paste bug.)

`assignRunway` (comments **FR2.1 / FR2.3**):

| Type | Preferred strip | Backup |
|------|-----------------|--------|
| EMERGENCY | `RWY_C` | Arrival `RWY_A`, departure `RWY_B` |
| CARGO | `RWY_C`; may **preempt** a commercial occupant and re-queue them | — |
| COMMERCIAL | Arrival `RWY_A`, departure `RWY_B` | Empty `RWY_C` overflow |

`generateAVN` / `displayAVNs` / `payAVN` (amount must be ≥ fine). `displayQueues` prints copies of the three sub-queues. `runwayToString` → `RWY_A` / `RWY_B` / `RWY_C`.

---

## `Simulation`

Members: `atc`, `pending` / `active` vectors, `duration = 300`, `sf::RenderWindow*`.

Tick (`start`):

1. Poll SFML close.
2. Release runways occupied ≥ 10 s (in addition to the runway thread).
3. Move `pending` → `active` when `scheduledTime <= now`; `setStatus(ACTIVE)`; `atc->addFlight`.
4. `scheduleFlights()`; each active flight `updateFuel`; if elapsed ≥ `getPhaseDuration()`, `progressPhase`; completed flights release the runway.
5. **`randomGroundFault()`** (5%): during TAXI or AT_GATE, tow off, release runway, mark COMPLETED.
6. Every 5 s of wall time: `displayFlights`, `displayQueues`, `displayRunwaysStatus`, `displayAVNs`.
7. `renderVisualization`; `sleep_for(1s)`.

String helpers: `flightPhaseToString`, `flightStatusToString`.

---

## Interactive `main` / `createFlight`

`createFlight`:

- 3-digit numeric id (e.g. `301`).
- Airline name must match a seeded `Airline`.
- Type from name: PIA/AirBlue commercial; FedEx/Blue Dart cargo; else emergency.
- Non-emergency: `N/S/E/W` → NORTH/SOUTH/EAST/WEST.
- Emergency: `A`/`D` → ARR/DEP.
- Time `HH:MM` applied to **today** via `localtime_r` / `mktime`.
- Prefix: PK, AB, FE, PA, BD, AM (AghaKhan).

`main`: Y/N add flights; loop until N; cargo check; `simulation.start()`; print `flightNumber @ time - runway`; `airlinePortal` (enter AVN id and PKR amount).

---

## SFML visualization

`renderVisualization`: black clear; load **`arial.ttf`** (failure prints `Error loading font` and returns). Rectangles for RWY_A/B/C at y = 100/200/300: **green** free, **red** occupied. Blue “ATC” tower. Circles for active flights: yellow emergency, magenta cargo, cyan commercial; x = `100 + phase * 50`.

Window: `sf::VideoMode(600, 500)`, `"AirControlX Simulation"`.

---

## File-by-file

| File | Role |
|------|------|
| `OS PROJECT_AIR LINE MANAGEMENT SYSTEM.cpp` | Entire program (all classes + `main`) |
| `OS_Project_Statement -SP2025.pdf` | Course statement (large binary) |
| `airplane-with-circle-flight-path.zip` | Artwork archive |
| `line-travel-elements-blackboard.jpg.crdownload` | Incomplete image download |

No `.sln` in the tree listing. No extra data files for flights (all interactive / in-code airlines).

---

## Build and run

C++11+ · `std::thread` / `mutex` / `atomic` · SFML Graphics/Window/System · **`arial.ttf`** next to the binary.

`formatTime` uses **`localtime_r`** (POSIX). MSVC typically needs `localtime_s` or a compatibility shim.

```bash
g++ -std=c++17 "OS PROJECT_AIR LINE MANAGEMENT SYSTEM.cpp" -o AirControlX \
  -lsfml-graphics -lsfml-window -lsfml-system -pthread
./AirControlX
```

1. `Add flight details? (Y/N)` — enter at least one **FedEx** or **Blue Dart** cargo plus any others.
2. Watch ~5 minutes of console + SFML (or close the window to stop the loop).
3. Final schedule; optional AVN payment.

Tune `SIMULATION_DURATION` at the top of the `.cpp`.

---

## Limitations

- `NorthTime` / `SouthTime` / `EastTime` / `WestTime` are unused; no timed auto-spawn.
- `generateAVN` is never called from `checkViolation` — portal often has **no AVNs** even when console prints “AVN issued”.
- Commercial `processQueue` failure path mutates the **cargo** queue.
- `localtime_r` is not portable to stock MSVC.
- Ground fault and 20% violation speeds are random; runs are not reproducible without seeding control.
- Single `.cpp` (~1000 lines); SFML + font required for the visual path.
- Media zip / `.crdownload` are unused by code.

**Extend:** call `generateAVN` when `checkViolation` fires; wire directional spawn intervals; MSVC time; persist AVNs to disk.

---

## Author

**Mohammad Rohaan** — 22I-2327  
[https://github.com/rohaan2802](https://github.com/rohaan2802)
