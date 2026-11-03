# Smart Dhakkan --- IoT-Based Smart Manhole Monitoring System

**Making cities smarter and safer through connected manhole
monitoring.**

Smart Dhakkan is an IoT-based manhole monitoring system designed to help
municipalities and infrastructure teams remotely monitor manhole
conditions. The system combines an ESP32-based sensor unit, MQTT
communication, a Node.js backend, and a live web dashboard.

It is designed to help identify conditions such as hazardous gas
accumulation, rising water levels, manhole-cover movement or opening,
and the location of deployed manholes.

> **Project status:** Prototype / active development. Hardware
> integration, firmware, backend, and dashboard capabilities may evolve
> as the project develops.

## Contents

-   [Overview](#overview)
-   [Key Features](#key-features)
-   [System Architecture](#system-architecture)
-   [Hardware](#hardware)
-   [Sensor Interface and Working](#sensor-interface-and-working)
-   [Web Application](#web-application)
-   [MQTT Communication](#mqtt-communication)
-   [Backend Data Flow](#backend-data-flow)
-   [Getting Started](#getting-started)
-   [Configuration and Security](#configuration-and-security)
-   [Calibration and Testing](#calibration-and-testing)
-   [Repository Structure](#repository-structure)
-   [Limitations and Safety Notes](#limitations-and-safety-notes)
-   [Future Scope](#future-scope)
-   [References](#references)

## Overview

Manholes are critical parts of urban drainage and sewer infrastructure.
Manual inspection can be time-consuming and may expose workers to
hazardous environments. Smart Dhakkan aims to provide remote visibility
into the condition of deployed manholes using a network of
sensor-equipped devices.

Each manhole unit uses an ESP32 microcontroller to collect sensor
readings and publish data to a cloud MQTT broker. A server-side
application receives the messages, maintains the latest state of each
manhole, stores selected readings, and streams updates to browser
dashboards.

The dashboard presents manhole locations on a map, live sensor readings,
alert classifications, and a visual representation of the cover's
orientation.

## Key Features

-   **Carbon monoxide monitoring:** Reads CO concentration through an
    MQ-7 sensor.
-   **Water-level monitoring:** Uses a water-pressure sensor to estimate
    water level and help identify possible flooding.
-   **Cover orientation monitoring:** Uses a BNO085 IMU to track
    orientation changes and help detect opening, tilting, or disturbance
    of the manhole cover.
-   **GPS location:** Uses a NEO-6M GPS module to provide latitude and
    longitude for each deployed unit.
-   **Live remote dashboard:** Displays current readings and manhole
    status through a browser.
-   **Geospatial visualization:** Plots manholes on an interactive
    Leaflet map.
-   **Alert classification:** Groups manhole conditions into Normal,
    Warning, and Critical states.
-   **Historical logging:** Stores selected sensor readings for later
    review.
-   **3D orientation visualization:** Uses Three.js to represent the
    physical unit's orientation.

## System Architecture

``` text
             MANHOLE UNIT
       +----------------------+
       |       ESP32          |
       |                      |
       |  MQ-7  - CO          |
       |  BNO085 - Orientation|
       |  Pressure - Water    |
       |  NEO-6M - GPS        |
       +----------+-----------+
                  |
             Wi-Fi / MQTT
                  |
                  v
          +---------------+
          | HiveMQ Cloud  |
          | MQTT Broker   |
          +-------+-------+
                  |
             MQTT Subscribe
                  |
                  v
       +----------------------+
       | Node.js + Express    |
       |                      |
       | JSON parsing         |
       | Live device state    |
       | Data persistence     |
       +----------+-----------+
                  |
             Server-Sent Events
                  |
                  v
       +----------------------+
       | Web Dashboard        |
       |                      |
       | HTML / CSS / JS      |
       | Leaflet.js map       |
       | Three.js model       |
       | Live alerts/readings |
       +----------------------+
```

### Technology Stack

  Layer                  Technology
  ---------------------- --------------------------
  Microcontroller        ESP32
  Firmware               Arduino framework / C++
  Connectivity           Wi-Fi
  Messaging              MQTT
  MQTT broker            HiveMQ Cloud
  Backend                Node.js, Express.js
  Live browser updates   Server-Sent Events (SSE)
  Database               SQLite
  Frontend               HTML, CSS, JavaScript
  Map                    Leaflet.js
  3D visualization       Three.js

## Hardware

The prototype's documented hardware interface includes the following
components:

  -----------------------------------------------------------------------
  Component               Purpose                 Interface
  ----------------------- ----------------------- -----------------------
  ESP32                   Reads sensors and       ---
                          publishes device data   

  MQ-7 gas sensor         Carbon monoxide sensing Analog

  BNO085 IMU              Manhole-cover           I²C
                          orientation and         
                          movement                

  Water-pressure sensor   Water-level estimation  Analog

  NEO-6M GPS              Latitude and longitude  UART (RX/TX)

  IRLZ44N MOSFET          Controls the MQ-7       ESP32 PWM / GPIO
                          heater cycle            

  20 kΩ and 10 kΩ         Voltage divider for 5 V Analog signal
  resistors               sensor outputs          conditioning
  -----------------------------------------------------------------------

**Important:** Confirm the exact pin assignments, supply requirements,
and wiring against the firmware and the specific breakout boards being
used. The GPIO numbers below are the documented example connections, not
a complete pin map.

## Sensor Interface and Working

### 1. ESP32 Microcontroller

The ESP32 is the main controller of each manhole unit. It:

1.  Reads sensor data.
2.  Processes the readings and device status.
3.  Connects to the configured Wi-Fi network.
4.  Connects to the MQTT broker.
5.  Publishes data to the topic assigned to that manhole ID.

Example topic:

``` text
manhole/DL_001/data
```

### 2. MQ-7 Carbon Monoxide Sensor

The MQ-7 is used to monitor carbon monoxide (CO) concentration.

-   The sensor's analog output (`AOUT`) is connected to an ESP32 ADC
    input; GPIO 34 is the documented example.
-   Since the sensor module may produce an output up to 5 V and ESP32
    GPIO inputs are not 5 V tolerant, a voltage divider is used.
-   The documented divider uses 20 kΩ and 10 kΩ resistors to scale the
    signal before it reaches the ESP32.
-   The MQ-7 requires a controlled heater cycle. The design uses an
    IRLZ44N logic-level MOSFET, driven by an ESP32 GPIO/PWM signal, to
    control the heater.
-   Readings must be taken during the appropriate low-heater phase,
    following the sensor datasheet and the actual module circuit.
-   Initial calibration is required under known reference conditions
    before estimating gas concentration in ppm.

The heater timing, ADC conversion, load resistance, calibration
constants, and ppm calculation must match the actual sensor circuit and
firmware. Do not assume that an uncalibrated raw ADC value is a reliable
ppm measurement.

### 3. BNO085 IMU

The BNO085 measures the orientation of the manhole unit.

-   Communicates with the ESP32 over I²C.
-   Provides orientation information that can be represented as roll,
    pitch, and yaw.
-   The system compares the current orientation with a stored baseline
    orientation.
-   Significant changes can be used as an indication of cover opening,
    tilting, or physical disturbance.

The baseline should be captured when the cover is in its known normal,
closed position. Thresholds should be validated in the actual
installation to avoid false alerts.

### 4. Water-Pressure Sensor

The water-pressure sensor is used to estimate water level.

-   Its analog output is read by an ESP32 ADC input.
-   If the selected sensor outputs up to 5 V, use a suitable voltage
    divider or signal-conditioning circuit before connecting it to the
    ESP32.
-   Calibrate the sensor with the sensing element exposed to air and no
    applied water pressure to establish its zero offset.
-   Convert calibrated pressure readings into water-level units using
    the sensor's range and the installation geometry.

The exact conversion depends on the selected pressure sensor, its output
characteristics, and how it is installed.

### 5. NEO-6M GPS Module

The NEO-6M GPS module provides the geographic location of the manhole
unit.

-   Communicates with the ESP32 over UART.
-   Supplies latitude and longitude when a valid GPS fix is available.
-   Location data is associated with the manhole ID and displayed on the
    dashboard map.

GPS reception can be poor underground or in covered locations. The
installation must be designed so the antenna can obtain a usable signal,
or the location should be configured by another method.

## Web Application

The web interface provides a centralized view of deployed manholes and
their latest reported conditions.

### Dashboard Capabilities

-   Live CO concentration display (ppm).
-   Live water-level display.
-   Manhole location plotting using GPS coordinates.
-   Normal, Warning, and Critical status classification.
-   Historical sensor-data logging.
-   Individual manhole detail panels.
-   3D orientation visualization of the manhole unit.

### Frontend

The frontend is built using HTML, CSS, and JavaScript.

**Leaflet.js** is used to display an interactive map. Each manhole is
represented by a custom SVG marker placed using its latitude and
longitude.

**Three.js** is used to display a 3D model of the manhole unit. The
model's orientation is updated using the IMU data, helping visualize
tilt and structural disturbance.

The dashboard includes an overview panel and individual manhole detail
panels for sensor readings, location, and status.

### Backend

The backend uses Node.js and Express.js. Its documented responsibilities
are:

1.  Connect to the MQTT broker using configured credentials.
2.  Subscribe to manhole data topics, including wildcard subscriptions
    where appropriate.
3.  Parse incoming JSON messages.
4.  Update the latest in-memory state for each manhole.
5.  Store selected readings in SQLite.
6.  Push live updates to connected browsers using Server-Sent Events
    (SSE).

### Why Server-Sent Events?

The dashboard primarily needs updates from the server to the browser.
SSE provides a lightweight, one-way streaming connection for this use
case, without requiring continuous browser polling or a bidirectional
WebSocket connection.

## MQTT Communication

Each manhole publishes its data using a topic containing its unique ID.

### Topic Format

``` text
manhole/<MANHOLE_ID>/data
```

Example:

``` text
manhole/DL_001/data
```

The backend can subscribe to all manhole data topics using an MQTT
wildcard subscription:

``` text
manhole/+/data
```

### Example Payload

The following is an **illustrative sensor payload** showing the kind of
fields the application can exchange. The exact field names and units
must be kept consistent between the ESP32 firmware and backend parser.

``` json
{
  "device": "ESP32",
  "manhole_id": "DL_001",
  "status": "ONLINE",
  "co_ppm": 34.0,
  "water_level_m": 0.00,
  "roll": 0.0,
  "pitch": 0.0,
  "yaw": 0.0,
  "latitude": 28.581108,
  "longitude": 77.321061,
  "uptime_ms": 1712143,
  "wifi_rssi": -56
}
```

A basic connectivity-test message may contain device status, message
count, uptime, free heap, Wi-Fi RSSI, and IP address before all sensors
are connected. The backend should validate incoming fields and handle
missing or unavailable sensor values safely.

## Backend Data Flow

``` text
ESP32 publishes JSON
        |
        v
MQTT broker receives message
        |
        v
Node.js backend receives MQTT event
        |
        v
Parse and validate JSON
        |
        +----------------------+
        |                      |
        v                      v
Update live state       Store selected readings
        |                      |
        v                      |
Send update over SSE           |
        |                      |
        v                      |
Browser receives event         |
        |                      |
        v                      |
Update map, panels,             |
3D orientation and alerts <-----+
```

## Getting Started

The exact commands and environment variables depend on the repository's
final folder structure and implementation. The following is a general
setup outline.

### Prerequisites

-   Arduino IDE with ESP32 board support.
-   ESP32 development board and the required sensors.
-   A Wi-Fi network accessible to the ESP32.
-   A HiveMQ Cloud MQTT broker account or another compatible MQTT
    broker.
-   Node.js and npm.
-   A modern web browser.

### 1. Clone the Repository

``` bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd <YOUR_REPOSITORY_FOLDER>
```

### 2. Configure ESP32 Firmware

1.  Open the ESP32 firmware sketch in Arduino IDE.
2.  Install the libraries required by the sketch.
3.  Set the Wi-Fi and MQTT configuration.
4.  Verify the sensor pin assignments and wiring.
5.  Select the correct ESP32 board and port.
6.  Upload the firmware.
7.  Open Serial Monitor and verify Wi-Fi and MQTT connection messages.

### 3. Configure the Backend

Navigate to the backend directory and install its dependencies:

``` bash
cd backend
npm install
```

Create a local environment configuration file (for example, `.env`)
using the variable names expected by the backend. Typical settings
include:

``` env
MQTT_HOST=<YOUR_MQTT_BROKER_HOST>
MQTT_PORT=<YOUR_MQTT_BROKER_PORT>
MQTT_USERNAME=<YOUR_MQTT_USERNAME>
MQTT_PASSWORD=<YOUR_MQTT_PASSWORD>
```

Start the backend using the script defined in its `package.json`, for
example:

``` bash
npm start
```

### 4. Run the Frontend

If the frontend is served by the Express backend, open the configured
local application URL in a browser. If it is hosted separately, follow
the frontend's own development or deployment instructions.

> The commands above are a setup template. Update directory names,
> environment-variable names, ports, and start commands to match the
> actual repository before publishing this README as a verified
> installation guide.

## Configuration and Security

-   Do not commit Wi-Fi passwords, MQTT usernames, MQTT passwords,
    broker certificates, API keys, or other secrets.
-   Keep credentials in local configuration files or environment
    variables.
-   Add `.env` and other secret-bearing files to `.gitignore`.
-   Use TLS-enabled MQTT connections where supported by the broker and
    client configuration.
-   Validate incoming MQTT payloads before storing or displaying them.
-   Use unique IDs for deployed manholes.
-   Restrict broker permissions so each device can publish only to its
    authorized topic where the broker supports topic-level access
    control.

Example `.gitignore` entries:

``` gitignore
.env
.env.*
!.env.example
node_modules/
*.db
*.sqlite
.DS_Store
```

## Calibration and Testing

Before field deployment, test each sensor independently and then test
the complete data path.

  -----------------------------------------------------------------------
  Component                           Calibration / validation
  ----------------------------------- -----------------------------------
  MQ-7                                Follow the datasheet heater cycle;
                                      allow required preheating;
                                      calibrate using a suitable known
                                      reference

  BNO085                              Record the normal closed-cover
                                      orientation and validate
                                      tilt/opening thresholds

  Water-pressure sensor               Establish zero offset in air and
                                      validate against known water depths
                                      or pressures

  NEO-6M GPS                          Verify valid fixes and compare
                                      coordinates with a known location

  MQTT                                Confirm the device publishes to the
                                      correct manhole topic and the
                                      backend receives valid JSON

  SSE                                 Confirm browser clients receive
                                      updates without manual refresh

  Dashboard                           Confirm map position, sensor
                                      values, 3D orientation, and alert
                                      state update correctly
  -----------------------------------------------------------------------

## Repository Structure

A suggested repository layout is:

``` text
smart-dhakkan/
├── firmware/
│   └── esp32/
│       └── smart_dhakkan.ino
├── backend/
│   ├── server.js
│   ├── package.json
│   └── ...
├── frontend/
│   ├── index.html
│   ├── css/
│   ├── js/
│   └── ...
├── docs/
│   ├── images/
│   │   ├── dashboard-prototype.png
│   │   └── smart-dhakkan-poster.jpg
│   └── ...
├── .gitignore
└── README.md
```

This is a suggested layout, not a claim about the current repository
contents. Adjust it to match the actual files.

## Limitations and Safety Notes

-   This project is a prototype and should not be treated as a certified
    life-safety or environmental monitoring system.
-   Low-cost gas sensors such as the MQ-7 require correct heater
    operation, calibration, environmental compensation, and validation.
    They are not a substitute for certified gas detectors.
-   Do not enter a manhole based only on readings from this prototype.
    Confined-space entry requires established safety procedures and
    suitable certified monitoring equipment.
-   Sensor readings can be affected by placement, condensation,
    contamination, temperature, humidity, power quality, and calibration
    drift.
-   GPS may not work reliably underground or beneath a metal cover.
-   Alert thresholds must be selected and validated for the intended
    environment; example dashboard states are not universal safety
    limits.

## Future Scope

Potential improvements include:

-   Deploying and managing multiple manholes across a city or region.
-   Configurable alert thresholds and notification channels.
-   Improved sensor calibration and fault detection.
-   Device-health monitoring, including battery and connectivity status.
-   Secure device provisioning and per-device MQTT authorization.
-   More detailed historical charts and trend analysis.
-   Offline buffering and automatic recovery after network
    interruptions.
-   Field validation and enclosure design for long-term underground
    operation.

## References

### MQ-7 Carbon Monoxide Sensor

-   [MQ-7 sensor module
    reference](https://www.areresearch.net/2017/03/detect-co-with-mq-7-sensor-module.html)
-   [MQ-7 datasheet ---
    SparkFun](https://cdn.sparkfun.com/assets/b/b/b/3/4/MQ-7.pdf)

### Circuit Design

-   [Wokwi circuit
    diagram](https://wokwi.com/projects/455739634182512641)

### Water-Pressure Sensor

-   [Arduino water-pressure sensor project --- Electronic
    Clinic](https://www.electroniclinic.com/arduino-water-pressure-sensor-project-water-level-pressure-sensor/)

------------------------------------------------------------------------

## Project Identity

**Smart Dhakkan --- IoT-Based Manhole Monitoring System**

*Making Cities of Tomorrow Smarter and Safer.*

Developed by **DTECH**.
