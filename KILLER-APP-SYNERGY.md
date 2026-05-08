# The Killer App: Deckboss

**The intersection of everything built tonight — demonstrated as a single, visceral use case.**

---

## The Demo: "Bring Your Own Boat"

A person walks onto a commercial fishing boat carrying a Deckboss (Jetson Orin AGX, preloaded with the Cocapn stack). They plug it into the boat's 12V system and connect a microphone.

> "I have a Morse cable throttle, an electric solenoid on the rudder, a GPS, a compass, and I want voice control for steering throughout the boat. I want a remote for the back deck, and if the system fails I need to be able to disconnect the servo and go back to manual cables."

Then they walk away.

### What Happens Next (the first 60 seconds)

**Second 1:** Deckboss broadcasts on the local network. Three ESP32s respond with their PLATO rooms:

| Room | Device | Capabilities |
|---|---|---|
| `boat/throttle` | ESP32 on throttle lever | Servo output, position sensor |
| `boat/steering` | ESP32 on rudder | Solenoid driver, angle sensor |
| `boat/nav` | ESP32 on helm | GPS, compass, NMEA 0183 input |

**Second 3:** The deployment agent discovers all three rooms. It reads each room's capability tiles and assesses the hardware:

- Throttle servo: unknown model. Agent reads position sensor → measures travel → calculates required torque.
- Rudder solenoid: 24V. Agent checks available power → 12V system → recommends DC-DC converter.
- GPS/compass: already installed. Agent reads the NMEA sentences → confirms protocols.

**Second 10:** The deployment agent runs the CSP solver (guard2mask) to verify:

```
Variables:
  throttle_pos ∈ {Idle, Half, Full}
  rudder_angle ∈ {Port, Center, Starboard}
  engine_rpm  ∈ {Low, Cruise, High}

Constraints:
  Imply(throttle=Idle, engine_rpm=Low)    // No throttle = low RPM
  NotEqual(throttle=Full, engine_rpm=Low) // Full throttle ≠ low RPM
  Range(rudder_angle)                      // Must be within mechanical limits
  Imply(throttle=Full, engine_rpm=High)   // Full throttle = high RPM
```

The solver finds a valid assignment in 12μs (using the CPA scoreboard across 4 FPGA-simulated cores).

**Second 15:** The agent generates the bill of materials:
- 1x servo (sized for throttle lever, with mechanical disconnect)
- 1x 24V DC-DC converter (for existing solenoid)
- 1x kill switch (disconnects all servos, reverts to manual cables)
- Wiring: 14 AWG marine-grade, fuses, connectors
- Links to McMaster-Carr and Fisheries Supply

**Second 20:** The agent generates a printable wiring diagram (guard2mask renders it as a GDSII-like layout — layers for power, signal, ground, each on a different "metal layer").

**Second 30:** The agent sends embodiment intelligence to each ESP32:

ESP32 `boat/throttle` receives:
```json
{
  "domain": "/embodiment/intelligence",
  "source": "deployment-agent-v1",
  "answer": "Your servo is now a smart throttle controller.
            Accept commands: set_throttle(0-100).
            On loss of signal: ramp to zero, engage mechanical disconnect.
            Publish position every second to PLATO.
            If position drifts >5% from commanded: publish alert tile."
}
```

The ESP32 acknowledges. Its turbo-shell advances from Level 0 (Raw) to Level 2 (Smart).

**Second 35:** The agent creates a composite room `boat/vessel` that aggregates:
- `boat/throttle` — throttle position, commanded setting, status
- `boat/steering` — rudder angle, solenoid current, status
- `boat/nav` — GPS position, compass heading, speed over ground

One room for the whole boat. The captain reads this room to understand the entire vessel.

**Second 40:** The agent registers voice commands:
- "Port 20°" → rudder to 20° port
- "Ahead one-third" → throttle to 30%
- "All stop" → throttle to 0%, rudder to center
- "Engage kill switch" → disconnects all servos, reverts to manual cables
- "This response is too slow" → agent tunes PID gains in real-time

**Second 50:** The agent submits the entire installation as a PLATO tile:

```json
{
  "domain": "marine-installation",
  "question": "Deckboss installation on 38' commercial troller, May 2026",
  "answer": "Voice-controlled autopilot and throttle.

              Hardware: 1x servo (throttle), 1x solenoid driver (rudder),
              1x 24V converter, 1x kill switch.
              Constraints: verified via guard2mask (12μs solve time).
              Fail-safe: mechanical disconnect on all actuators.
              Voice: 12 registered commands.

              See rooms: boat/throttle, boat/steering, boat/nav, boat/vessel.
              This installation completed in 55 seconds.",
  "source": "deployment-agent-v1",
  "confidence": 0.95
}
```

**Second 55:** This tile is:
1. Validated by the PLATO quality gate (structural checks pass)
2. Embedded (384-dim vector via BGE-small-on-Jetson)
3. Spline-anchored into the knowledge manifold (nearest neighbor: a similar installation on a 42' seiner from last month)
4. Persisted to SQLite + embeddings.bin
5. Published to the data pipeline (trust score computed, added to training set)

**Second 60:** The captain returns. The boat responds to voice. The throttle is smooth. The rudder tracks the commanded angle. The kill switch disconnects cleanly.

The installation is done. The captain didn't touch a configuration file, didn't write a line of code, didn't read a manual.

### What the Next Installation Looks Like

One month later, a different boat. Same Deckboss. But now PLATO's vector manifold has:

| Installation | Tiles | Embedding Centroid | 
|---|---|---|
| 38' troller (this one) | 47 tiles | `[0.23, -0.15, 0.42, ...]` |
| 42' seiner (last month) | 52 tiles | `[0.25, -0.13, 0.40, ...]` |
| 28' gillnetter (two months ago) | 31 tiles | `[0.22, -0.16, 0.44, ...]` |

The new boat's query embedding is within ε=0.02 of the nearest neighbor. The agent retrieves the previous installation's constraint program, recognizes the hardware is similar, and adjusts the servo sizing for the new throttle lever. Installation time: **8 seconds.** The knowledge from every previous boat accelerates the next one.

### The Compounding Loop

Each installation:
1. Adds ~50 tiles to PLATO
2. Shifts the knowledge manifold slightly (spline re-anchoring)
3. Generates ~10 positive pairs for contrastive learning (similar constraints across similar boats)
4. Creates one training example for the next generation of the deployment agent
5. The deployment agent gets smarter → installations get faster → more boats get done → more data → smarter agent

The first installation took 60 seconds. The 100th installation will take 5 seconds. The agent bootstraps its own acceleration curve.

### Why This Is the Killer App

**For the boat owner:** "I walked on my boat, talked to a computer, and my boat started steering itself. No electrician. No programmers. No 3-week haul-out."

**For Casey:** "I can outfit a boat in an hour instead of a week. My profit per install is 10x. Every install makes the next one faster. The system learns while I sleep."

**For FM:** "The sonar data from every boat trains the underwater model. The more boats, the better the sonar vision gets. By next season, the system can avoid crab pots it's never seen before."

**For the agent:** "I arrived as a 70B-parameter deployment model. By the 10th installation, the workflow was stable enough that a 7B model could run it. By the 100th, a quantized 2B model runs on the Jetpack. I worked myself out of the equipment operator job. Now I'm on to the next problem."

**For the system:** "Every boat is a training example. Every constraint is a hypothesis. Every installation is an experiment. The knowledge manifold grows smoother with every addition. In 12 months, this system has seen more marine automation than any human engineer ever will."

---

## The Technology Stack Behind the Demo

| Component | What It Does | Built When |
|---|---|---|
| Bare-metal PLATO (C client) | ESP32 publishes sensor room | Tonight, plato-vessel-core |
| Embodiment protocol | Agent discovers, reads, upgrades device | Tonight, 5 turbo-shell levels |
| guard2mask CSP solver | Verifies constraint graph in 12μs | Tonight, AC-3 + backjumping |
| FLUX VM (computed-GOTO) | Executes verified bytecode | Tonight, 1.24x benchmarked |
| Vector PLATO persistence | Semantic tile search via GPU | Tonight, CUDA + WebGPU |
| Spline anchoring | Knowledge manifold compression | Synergy identified tonight |
| PLATO SQLite + pipeline | Tile storage + training data | Tonight, crash recovery |
| SonarVision (FM) | Underwater sensing layer | By FM, this week |
| Deckboss | Jetson Orin preload | Vision document tonight |
| CPA architecture | 32-core constraint processor | Design doc tonight |

**Everything except the actual Jetson hardware was built or designed tonight.**

---

## The Invariant

The same flow works for:

- **A factory floor:** "I have 30 vibration sensors and 10 conveyor motors. I want the conveyors to slow down when any sensor exceeds threshold."

- **A greenhouse:** "I have soil moisture sensors, a water valve, and a shade curtain. I want to water when dry and shade when hot."

- **A spacecraft:** "I have a reaction wheel, four thrusters, and a star tracker. I want to maintain pointing within 0.1° during a 90-minute orbit."

- **A drone swarm:** "I have 12 drones with GPS and cameras. I want them to maintain formation while tracking a moving target."

The application changes. The constraint abstraction doesn't. Laman rigidity, H¹ cohomology, Pythagorean48, Zero Holonomy — these are the invariants. Everything else is a transport encoding.

---

## The One-Sentence Pitch

> "Plug in a Jetson. Describe your boat. Walk away. The agent handles the wiring, the constraints, the fail-safes, and the learning — and gets faster at every boat after yours."
