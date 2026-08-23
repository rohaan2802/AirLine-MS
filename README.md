# AirLineHandlingSystem

**AirControlX**-style airline / ATC simulation for an OS course project. Models flights, runways, priority queues, speed violations (**AVNs**), fuel emergencies, multithreading, and an SFML status window. Main source: `OS PROJECT_AIR LINE MANAGEMENT SYSTEM.cpp`. Accompanying statement: `OS_Project_Statement -SP2025.pdf`.

## Overview

The program simulates airport operations over a fixed duration (**300 seconds** / 5 minutes) with directional schedules:

| Direction | Meaning | Spawn interval (constants) |
|-----------|---------|----------------------------|
| NORTH | International arrival | 180s |
| SOUTH | Domestic arrival | 120s |
| EAST | International departure | 150s |
| WEST | Domestic departure | 240s |
| ARR / DEP | Emergency arrival/departure | as needed |

Airlines seeded in-code: PIA, AirBlue (commercial); FedEx, Blue Dart (cargo); PAF, AghaKhan Air (emergency).

Three runways (`RWY_A`, `RWY_B`, `RWY_C`) each run a **worker thread** protected by a `mutex`. ATC holds flight queues under another mutex and issues AVNs when phase speed limits are broken.

## Features

- Aircraft types: `COMMERCIAL`, `CARGO`, `EMERGENCY`  
- Flight phases: Holding → Approach → Landing → Taxi → Gate / Takeoff roll → Climb → Cruise  
- Per-phase fuel burn and speed generation (including intentional violation chance)  
- Low fuel promotes a flight to emergency  
- Runway assignment and threaded occupancy  
- AVN records with pay-via-portal helper (`airlinePortal`)  
- Interactive flight creation (`createFlight`) before sim start; **at least one cargo flight required**  
- SFML window `600×500` titled “AirControlX Simulation” for visualization ticks  
- Console dumps of queues, runway status, AVNs, and final schedule  

## Tech stack

| Component | Technology |
|-----------|------------|
| Language | C++11+ |
| Concurrency | `std::thread`, `std::mutex`, `std::atomic`, `chrono` |
| Graphics | SFML Graphics |
| Containers | `vector`, `queue` |

## Project structure

```
AirLineHandlingSystem/
├── OS PROJECT_AIR LINE MANAGEMENT SYSTEM.cpp   # all classes + main
├── OS_Project_Statement -SP2025.pdf
├── airplane-with-circle-flight-path.zip        # media assets
└── line-travel-elements-blackboard.jpg.crdownload
```

Major types: `Airline`, `Flight`, `Runway`, `AVN`, `AirTrafficControl`, `Simulation`.

## How to build / run

### Requirements

- C++ compiler with pthread/`std::thread` support  
- SFML 2.x  
- Prefer Linux/macOS or MinGW for `localtime_r` (used in `formatTime`); on MSVC you may need a small port to `localtime_s`

### Example

```bash
g++ -std=c++17 "OS PROJECT_AIR LINE MANAGEMENT SYSTEM.cpp" -o AirControlX \
  -lsfml-graphics -lsfml-window -lsfml-system -pthread
./AirControlX
```

## Usage

1. Run the binary.  
2. Answer **Y/N** to add flights; enter details when prompted (airline, direction, schedule, etc.).  
3. Ensure at least one **cargo** flight exists or the program exits with an error.  
4. Simulation runs for ~5 minutes of sim logic (1s sleep ticks), printing updates and drawing the SFML view.  
5. Afterward, review the final schedule and use the airline portal to list/pay AVNs.

## How to extend / modify

- Tune `SIMULATION_DURATION` and direction intervals at the top of the file.  
- Add airlines to the `airlines` vector.  
- Expand `renderVisualization()` for richer runway/flight graphics.  
- Persist AVNs to disk; add a fuller airline portal menu.  
- Align behavior with requirements in `OS_Project_Statement -SP2025.pdf`.

## Author

**rohaan2802** — [https://github.com/rohaan2802](https://github.com/rohaan2802)
