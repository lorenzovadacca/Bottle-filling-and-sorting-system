# Bottle filling and sorting system (Fluid Automation)

A complete simulation, electro-pneumatic design, PLC program implementation, and interactive HMI development for an automated industrial beverage filling and sorting line, developed for the **Fluid Automation** course (Master's Degree in Mechatronics Engineering) at Politecnico di Torino.

---

## Table of contents
1. [Overview](#overview)
2. [Requirements](#requirements)
3. [Architecture](#architecture)
   - [Pneumatic & Electro-Pneumatic System](#1-pneumatic--electro-pneumatic-system)
   - [PLC Control Logic & POUs Architecture](#2-plc-control-logic--pous-architecture)
   - [OPC Interfacing & Real-Time Animation](#3-opc-interfacing--real-time-animation)
   - [Human-Machine Interface (HMI)](#4-human-machine-interface-hmi)
4. [Specifications](#specifications)
   - [Digital inputs](#digital-inputs)
   - [Digital outputs](#digital-outputs)
   - [Analog inputs](#analog-inputs)
5. [Results](#results)

---

## Overview

This project models and automates an industrial plant that fills, caps, and sorts bottles of three beverage types (Coca-Cola, Fanta, and Pepsi) based on operator selection:
- **Bottle Infeed & Transport:** Empty bottles from an input buffer are pushed onto the primary transfer conveyor by pneumatic cylinder A.
- **Filling Station:** Bottles are transported to the filling head beneath the intermediate dosing tank; the conveyor stops while fluid is metered and dispensed.
- **Beverage Storage & Dosing:** Three independent reservoir tanks (Coca-Cola, Fanta, Pepsi) feed a centralized dispensing tank. Automated level checks initiate refill whenever a tank falls below the threshold for 6 bottles.
- **Corking / Capping Station:** Once filled, the bottle is moved along the belt to a capping position where pneumatic cylinder B extends to seal the bottle.
- **Sorting & Palletizing Buffer:** A dedicated accumulator collects completed bottles of the same beverage type. When a buffer reaches its target batch capacity (12 bottles), pneumatic pusher cylinders transfer the batch to dedicated output conveyors.
- **Safety & Homing:** Integrated emergency stop handling, full line homing, and automatic resumption from interrupted cycle steps upon emergency clear and start.

---

## Requirements

- **PLC Programming & HMI Development:** **CODESYS V3.5** (IEC 61131-3 compliant programming environment).
- **Physical & Pneumatic Simulation:** **Festo Didactic FluidSIM 5 / FluidSIM Pneumatics**.
- **Data Exchange Middleware:** **Festo Didactic EzOPC** (OPC DA server/client protocol enabling synchronous bidirectional exchange between CODESYS and FluidSIM).

---

## Architecture

### 1. Pneumatic & Electro-pneumatic system
The plant is simulated in FluidSIM and comprises distinct pneumatic actuation sub-circuits:
- **Infeed Cylinder A:** Double-acting pneumatic cylinder actuated by a $5/2$-way directional solenoid valve pushing empty bottles into the system.
- **Capping Cylinder B:** Double-acting cylinder driven by a $5/2$-way solenoid valve simulating downward bottle corking/pressing.
- **Sorting Pushers P1 & P2:** Pneumatic cylinders operated via dual-solenoid $5/3$-way directional valves (`P1+`/`P1-` and `P2+`/`P2-`) for bi-directional routing and sorting across buffer stations.
- **Beverage Dosing & Cut-off Valves:** Solenoid valves controlling flow from the storage tanks to the central tank (`V_fanta1`, `V_fanta2`, `V_cocacola1`, `V_cocacola2`, `V_pepsi1`, `V_pepsi2`) and discharge into the bottle (`Valve`).
- **Speed & Deceleration Regulation:** Meter-out variable one-way throttle valves to ensure smooth extension/retraction strokes.

### 2. PLC Control Logic & POUs Architecture
The control program in CODESYS is partitioned into structured Program Organization Units (POUs) using multiple IEC 61131-3 languages:
- **`Main` (Ladder Diagram - LD):** Oversees master power management (`ONOFF`), Start/Stop memory latches (`M_Start`, `M_Stop`), emergency triggers, batch size initialization (`MOVE` function blocks), production targets (`ADD`), and countdown tracking (`CTD`).
- **`CycleManagement` (Structured Text - ST):** Orchestrates sequential state transitions:
  - `STEP_00`: Verifies initial homing conditions ($A_0, B_0, P_0$), tank availability, and awaits `M_START`.
  - `STEP_0`: Cylinder A extends, feeding a bottle onto Conveyor 1.
  - `STEP_1` $\rightarrow$ `STEP_2`: Bottle reaches filling sensor; conveyor stops; bottom valve opens for liquid dispensing.
  - `STEP_2_1` $\rightarrow$ `STEP_3`: Dispensing completes (`TON` timer); conveyor transfers bottle to capping station; Cylinder B extends.
  - `STEP_4` $\rightarrow$ Sorting Steps (`STEP_PEPSI`, `STEP_COCA1`/`COCA2`, `STEP_FANTA1`/`FANTA2`): Identifies liquid type, triggers directional conveyors and pusher cylinders (P1/P2) into designated buffers, and re-triggers initial step `STEP_00`.
- **`Actions` (Ladder Diagram - LD):** Executes physical actuator commands for each active step, driving conveyor run coils, valve coils, filling timers (`TON`/`TOF`), and pusher solenoids.
- **`Emergency` (Ladder Diagram - LD):** Halts all active conveyor motors and closing valves immediately upon trip, sounds the emergency buzzer, flashes alarm lights, and enables resume logic upon reset.
- **`AnimationFC` (Ladder Diagram / Function Block Diagram):** Normalizes FluidSIM analog displacement signals ($0 \div 32761$) to pixel coordinates (`Real`) to drive smooth graphical movement in the visualization.
- **`Visibility` (Structured Text - ST):** Dynamic display management showing empty bottle queues, specific bottle textures (Fanta, Coca-Cola, Pepsi), level bars, and individual LED indicators ($1 \div 12$) inside the sorting buffers.

### 3. OPC Interfacing & Real-Time Animation
Real-time coupling via **EzOPC** synchronizes process states between CODESYS variables and FluidSIM components:
- Digital outputs from CODESYS (`AB10`, `AB11`, `AB12`) write directly to solenoid valves, electric motors, and pilot lights in FluidSIM.
- Discrete process sensors and limit switches (`EB10`, `EB11`) are mapped to CODESYS inputs.
- Continuous position transducers (linear potentiometers $0 \div 10\text{ V}$) stream analog stroke values (`ANIN21`, `ANIN23`) into CODESYS word registers.

### 4. Human-Machine Interface (HMI)
Developed in CODESYS Visualization:
- **Supervisory Controls:** Start, Stop, Emergency mushroom pushbutton, master ON/OFF selector switch.
- **Production Selection:** Numeric input keypad to set desired bottle production quantities per beverage type.
- **Dynamic 2D Graphics:** Visual representation of fluid levels in tanks, bottle travel across conveyors, cylinder strokes, and dynamic buffer palletization.
- **Status Dashboard:** Running/Stop pilot lights, emergency warning stack lights, acoustic buzzer, and tank filling gauges ($0 \div 100\%$).

---

## Specifications

### Digital inputs

| Card Address | Symbolic Tag | Hardware Component | Functionality |
| :--- | :--- | :--- | :--- |
| `EB10.OUT4` | `S_filling` | Optical Proximity Sensor | Bottle presence at filling station |
| `EB10.OUT5` | `S_capping` | Optical Proximity Sensor | Bottle presence at corking station |
| `EB10.OUT6` | `P0` | Optical Proximity Sensor | Bottle presence at initial transfer |
| `EB10.OUT7` | `end` | Optical Proximity Sensor | Bottle arrival at primary conveyor end |
| `EB11.OUT0` | `a0` | NO Limit Switch / Reed | Cylinder A retracted (home) |
| `EB11.OUT1` | `a1` | NO Limit Switch / Reed | Cylinder A fully extended |
| `EB11.OUT2` | `b0` | NO Limit Switch / Reed | Cylinder B retracted (home) |
| `EB11.OUT3` | `b1` | NO Limit Switch / Reed | Cylinder B fully extended (capping reached) |
| `EB11.OUT4` | `S_cocacola` | Optical Proximity Sensor | Coca-Cola buffer entry detection |
| `EB11.OUT5` | `S_pepsi` | Optical Proximity Sensor | Pepsi buffer entry detection |
| `EB11.OUT6` | `S_fanta` | Optical Proximity Sensor | Fanta buffer entry detection |

### Digital outputs

| Card Address | Symbolic Tag | Actuator / Indicator | Description |
| :--- | :--- | :--- | :--- |
| `AB10.IN0` | `A+` | Solenoid Valve Coil | Cylinder A extension (bottle push) |
| `AB10.IN1` | `B+` | Solenoid Valve Coil | Cylinder B extension (capping stroke) |
| `AB10.IN2` | `Conveyor 1` | Electric Motor Drive | Infeed bottle conveyor run command |
| `AB10.IN3` | `Conveyor 2` | Electric Motor Drive | Main process conveyor run command |
| `AB10.IN4` | `Conveyor 3` | Electric Motor Drive | Sorting output conveyor run command |
| `AB10.IN5` | `P1+` | Solenoid Valve Coil | Pusher 1 forward stroke |
| `AB10.IN6` | `P1-` | Solenoid Valve Coil | Pusher 1 return stroke |
| `AB10.IN7` | `Valve` | Solenoid Valve Coil | Main dispensing tank outlet valve |
| `AB11.IN0` | `V_fanta1` | Solenoid Valve Coil | Fanta reservoir feed line valve 1 |
| `AB11.IN1` | `V_fanta2` | Solenoid Valve Coil | Fanta reservoir feed line valve 2 |
| `AB11.IN2` | `V_cocacola1`| Solenoid Valve Coil | Coca-Cola reservoir feed line valve 1 |
| `AB11.IN3` | `V_cocacola2`| Solenoid Valve Coil | Coca-Cola reservoir feed line valve 2 |
| `AB11.IN4` | `V_pepsi1` | Solenoid Valve Coil | Pepsi reservoir feed line valve 1 |
| `AB11.IN5` | `V_pepsi2` | Solenoid Valve Coil | Pepsi reservoir feed line valve 2 |
| `AB11.IN6` | `P2+` | Solenoid Valve Coil | Pusher 2 forward stroke |
| `AB11.IN7` | `P2-` | Solenoid Valve Coil | Pusher 2 return stroke |
| `AB12.IN0` | `ONOFF_led` | Pilot Light (Green) | System power enabled indication |
| `AB12.IN1` | `start_led` | Pilot Light (Green) | Process operating indication |
| `AB12.IN2` | `stop_led` | Pilot Light (Red) | Process stopped / idle indication |
| `AB12.IN3` | `emergency_led` | Beacon Light (Red) | Emergency active visual alarm |
| `AB12.IN4` | `emergency_buz` | Acoustic Buzzer | Emergency active audible alarm |

### Analog inputs

| Card Address | Symbolic Tag | Component Type | Signal Range | Functionality |
| :--- | :--- | :--- | :--- | :--- |
| `ANIN21.OUT1` | `P1` | Displacement Encoder | $0 \div 10\text{ V}$ ($0 \div 32761$) | Pusher 1 position feedback |
| `ANIN21.OUT2` | `P2` | Displacement Encoder | $0 \div 10\text{ V}$ ($0 \div 32761$) | Pusher 2 position feedback |
| `ANIN23.OUT1` | `P_A` | Displacement Encoder | $0 \div 10\text{ V}$ ($0 \div 32761$) | Cylinder A stroke position feedback |
| `ANIN23.OUT2` | `P_B` | Displacement Encoder | $0 \div 10\text{ V}$ ($0 \div 32761$) | Cylinder B stroke position feedback |

---

## Results

Integrated co-simulation across FluidSIM and CODESYS verified full operational compliance:
1. **Accurate Dosing & Sorting Cycles:** Batch selections were executed with zero beverage cross-contamination. Conveyors halted reliably under filling and corking positions based on optical sensor triggers.
2. **Buffer Capacity Management:** Up-counters tracked accumulating bottles up to target thresholds ($12\text{ bottles/beverage}$), reliably triggering pusher strokes and clearing counters once containers exited.
3. **Emergency Interlocking & Recovery:** Emergency button actuation cut power to all actuator coils and motor drives instantly, while preserving batch counts and sequencer steps. Releasing the mushroom switch and pressing Start resumed the production sequence smoothly without requiring cycle re-initialization.
4. **HMI Visual Responsiveness:** Real-time scaling of analog stroke values successfully generated seamless animations of bottle translation, fluid filling levels, and buffer illumination indicators in the CODESYS dashboard.
