# AirLineHandlingSystem (AirControlX)

OS project: **airport / ATC simulation** with flights, **three runways**, **priority queues**, speed-limit **AVNs**, fuel emergencies, **C++ threads**, and a small **SFML** status window.

**Main:** `OS PROJECT_AIR LINE MANAGEMENT SYSTEM.cpp`  
**Spec:** `OS_Project_Statement -SP2025.pdf`  
[rohaan2802](https://github.com/rohaan2802)

---

## Table of contents

1. [Sim clock and spawn rates](#sim-clock-and-spawn-rates)
2. [Airlines and types](#airlines-and-types)
3. [Flight phases](#flight-phases)
4. [Runways and threads](#runways-and-threads)
5. [AVNs and portal](#avns-and-portal)
6. [Build and run](#build-and-run)

---

## Sim clock and spawn rates

Duration **300 s** (5 minutes). Directional schedules:

| Direction | Meaning | Interval constant |
|-----------|---------|-------------------|
| NORTH | International arrival | 180 s |
| SOUTH | Domestic arrival | 120 s |
| EAST | International departure | 150 s |
| WEST | Domestic departure | 240 s |
| ARR / DEP | Emergencies | as needed |

Interactive `createFlight` **before** start. **At least one cargo flight is required** or the program errors out.

---

## Airlines and types

Seeded in-code:

| Kind | Examples |
|------|----------|
| Commercial | PIA, AirBlue |
| Cargo | FedEx, Blue Dart |
| Emergency | PAF, AghaKhan Air |

Types: `COMMERCIAL`, `CARGO`, `EMERGENCY`.

---

## Flight phases

Holding → Approach → Landing → Taxi → Gate / Takeoff roll → Climb → Cruise  

Per-phase **fuel burn** and **speed** generation (including a chance of **intentional speed violation**). **Low fuel promotes emergency**.

---

## Runways and threads

Runways `RWY_A`, `RWY_B`, `RWY_C` — each a **worker thread** + `mutex`. ATC queues under another mutex. Occupancy while a flight uses the strip.

SFML window **600×500**, title **AirControlX Simulation**, visualization ticks. Console dumps queues, runway status, AVNs, final schedule. ~1 s sleep per sim tick.

Types: `Airline`, `Flight`, `Runway`, `AVN`, `AirTrafficControl`, `Simulation`.

Media: `airplane-with-circle-flight-path.zip`, leftover `.crdownload` image.

---

## AVNs and portal

Speed violations in a phase generate **AVN** records. Helper `airlinePortal` lists/pays them (extend to persist to disk).

---

## Build and run

C++11+ · `std::thread` / `mutex` / `atomic` / `chrono` · SFML Graphics  

`formatTime` uses `localtime_r` — Linux/macOS/MinGW; MSVC may need `localtime_s`.

```bash
g++ -std=c++17 "OS PROJECT_AIR LINE MANAGEMENT SYSTEM.cpp" -o AirControlX \
  -lsfml-graphics -lsfml-window -lsfml-system -pthread
./AirControlX
```

1. Y/N add flights; enter airline, direction, schedule.  
2. Ensure ≥1 cargo.  
3. Watch ~5 minutes of sim + SFML.  
4. Final schedule + portal.

Tune `SIMULATION_DURATION` and intervals at the top of the file. Align with the PDF statement.

---

## Author

**rohaan2802** · [https://github.com/rohaan2802](https://github.com/rohaan2802)
