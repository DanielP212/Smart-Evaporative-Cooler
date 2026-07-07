# Smart Evaporative Cooler using ESP32

## 📌 Project Overview
This repository contains the firmware and hardware architecture design for an **Automated Evaporative Cooling System**. The project is designed as an independent hardware-software co-design application, bridging low-level embedded software constraints with environmental and thermodynamic physical systems.

The core objective is to regulate perceived temperature (Heat Index) in real-time by controlling an ultrasonic mist maker (piezoelectric transducer) and an adjustable DC fan based on environmental data collected via a DHT22 sensor.

---

## 🛠️ Component Bill of Materials (BOM) - Engineering Specifications

| Category | Component (Model Suggestion) | Qty | Engineering Notes and Specifications |
| :--- | :--- | :--- | :--- |
| **Processing** | ESP32 (NodeMCU / ESP-WROOM-32) | 1 | Provides RTOS, hardware PWM, and future capability for Wi-Fi/Bluetooth. |
| **Sensor** | DHT22 (AM2302) | 1 | More reliable with a wider temperature and humidity range than the lower model (DHT11). |
| **Actuator (Air)** | DC Fan (12V) | 1 | Computer-type fan (80mm or 120mm). A 3 or 4-pin version facilitates RPM reading, but a 2-pin version works perfectly for PWM control. |
| **Actuator (Water)** | Ultrasonic Nebulizer Module (5V) | 1 | Piezoelectric module (usually sold as a disc + small control board). Creates cold mist. |
| **Power Control** | Logic-Level N-Channel MOSFET (IRLZ44N) | 2 | Essential. Acts as a high-power electronic switch controlled by the ESP32's 3.3V. (One for the fan, another for the nebulizer board). |
| **Protection** | Flyback Diode (1N4007) | 1 | Mandatory. Placed in parallel with the fan to absorb voltage spikes that can burn the MOSFET or ESP32 when turning off the motor. |
| **Power Supply** | Power Supply (12V DC, >2A) | 1 | Wall adapter. 2 Amperes ensure enough headroom to start the fan. |
| **Regulation** | DC-DC Buck Converter (LM2596) | 1 | Steps down the 12V from the main power supply to 5V, creating a safe power rail for the ESP32 (VIN pin) and the nebulizer. |
| **Passives** | Resistor Kit (1/4W) | Various | You will need a 10kΩ resistor (pull-up for the DHT22), and 330Ω and 10kΩ resistors to connect to the gate and GND of the MOSFETs (microcontroller protection). |
| **Prototyping** | Breadboard | 1 or 2 | For assembling and testing the circuit without soldering. |
| **Prototyping** | Jumper Wire Kit (M-M, M-F, F-F) | 1 | Essential wiring to interconnect the modules and the breadboard. |


---

## 💻 Software Architecture (Firmware Design)

The firmware is structured using **modular C/C++** over the Espressif IoT Development Framework (ESP-IDF) or FreeRTOS on Arduino core. It focuses on deterministic behavior, low-latency sensor polling, and precise actuator control.

### Core Features & Implementations:
1.  **Multi-tasking with FreeRTOS:**
    *   `Task_SensorPolling` (Core 1): Non-blocking reading of the DHT22 sensor using specialized hardware microsecond delays.
    *   `Task_ControlLogic` (Core 0): Executes the feedback loop and finite state machine (FSM).
    *   `Task_SafetyMonitor` (Core 0/1): High-priority watchdog to shut down actuators if anomalies (e.g., sensor disconnection, critical humidity levels) are detected.
2.  **Hardware PWM Control (LEDC Peripheral):**
    *   The DC Fan speed is dynamic, adjusted using **Pulse Width Modulation (PWM)** mapping to match the calculated thermal deviation.
3.  **Interrupts & Timers:**
    *   Hardware timers for deterministic execution intervals of the control loop.

---

## 📐 Control Logic & Feedback Loop

The system operates based on a closed-loop control system. Instead of simple thresholds, it evaluates the **Heat Index (HI)** to determine the optimal balance between mist generation and fan speed.




```
              +-------------------+
              |   DHT22 Sensor    |
              +---------+---------+
                        |
                        | (Temp & Humidity Data)
                        v
              +---------+---------+
              |  ESP32 Controller | <---+ Safety Watchdog
              | (FSM / Control)   |
              +---------+---------+
                        |
        +---------------+---------------+
        | (PWM Signal)                  | (ON/OFF or Duty Cycle)
        v                               v
+---------+---------+           +---------+---------+
|    DC Fan Speed   |           |  Ultrasonic Mist  |
|    Regulation     |           |     Generator     |
+-------------------+           +-------------------+

```

### State Machine / Algorithm:
*   **IDLE State:** Perceived temperature is optimal. Fan and Mist Maker are OFF (Power-saving mode).
*   **VENTILATION State:** Temperature is slightly high, but humidity is high. Only the Fan is active (Variable PWM) to generate airflow without oversaturating the air.
*   **COOLING State:** Temperature is high and humidity is low/moderate. **Mist Maker is activated** to promote evaporative cooling, and the **Fan is driven at proportional speed** to expel and direct the cooled air stream towards the body.
*   **CRITICAL State:** System shutdown if humidity reaches 100% (to prevent condensation on electronics) or sensor read failure.

---

## 🚀 Roadmap

## Phase 0: Planning and Theoretical Design
* **Requirement Definition:** Establish the exact Heat Index thresholds required for thermal comfort.
* **Datasheet Analysis:** Read the technical documentation for the ESP32, DHT22, and IRLZ44N MOSFET to confirm voltage and current limits.
* **Electrical Schematic Design (Draft):** Map out all connections, GPIOs, and isolation requirements on paper or using software (KiCad/Fritzing).
* **Material Acquisition (BOM):** Order all components from the Bill of Materials and ensure the arrival of protective/testing equipment (such as a multimeter).
* **Safety Planning:** Define protocols for handling water (from the mist maker/nebulizer) in close proximity to electronics and 12V currents.

## Phase 1: Architecture and Electricity (The Foundation)
* **Power Configuration:** Connect the 12V power supply to the LM2596 Buck converter and fine-tune the output to exactly 5V using the multimeter.
* **Breadboard Prototyping:** Assemble the base circuit without connecting the microcontroller, validating the power flow.
* **Isolation Testing (MOSFET):** Test the fan driving circuit by manually applying 3.3V to the gate of the MOSFET.
* **Protection Verification:** Confirm the correct installation of the Flyback diode in parallel with the fan motor.

## Phase 2: Software Infrastructure
* **Environment Setup:** Install the PlatformIO extension (VS Code) or ESP-IDF and initialize the Git repository.
* **Sensor Driver (DHT22):** Write the sensor polling code and validate the Temperature and Humidity printouts on the serial monitor.
* **PWM Testing (LEDC):** Configure the ESP32 hardware timer and validate the dynamic control of the fan speed.
* **Nebulizer Activation:** Test the digital pin (GPIO) that activates the piezoelectric module's control board.

## Phase 3: Control Logic and RTOS
* **FreeRTOS Integration:** Create independent tasks (`Task_SensorPolling` and `Task_ControlLogic`) assigned to cores 0 and 1 of the ESP32.
* **Memory Protection:** Implement queues or mutexes for the safe sharing of sensor data between parallel tasks.
* **Heat Index Calculation:** Program the mathematical function that fuses temperature and humidity into a single thermal reference value.
* **Finite State Machine (FSM):** Develop the core transition logic between the `IDLE`, `VENTILATION`, `COOLING`, and `CRITICAL` states.

## Phase 4: Physical Integration and Final Product
* **Bench Testing and Calibration:** Simulate extreme heat/humidity environments near the sensor to observe and fine-tune the FSM transitions.
* **PCB Transition:** Solder components onto a perfboard or design and order a custom PCB.
* **Enclosure Design (CAD):** Design the 3D model of the external enclosure, ensuring a watertight separation between the water reservoir and the electronics.
* **3D Printing and Assembly:** Print the structure, install the hardware, mount the boards, and seal the wet zones.
* **Thermodynamic Testing:** Operate the system in a real-world environment for several hours, adjusting the code's hysteresis cycle as needed.

