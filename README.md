# TECHNICAL MEMORANDUM: WRO FUTURE ENGINEERS

## 1. OUR PROJECT

* **Project Title:** THE BEST FAILURES
* **Category:** WRO Future Engineers
* **Team Name:** Colegio Nuestra Señora de las Mercedes
* **Members:** Sergio, Isaac, and Martín
* **School:** Colegio Nuestra Señora de las Mercedes (Herencia, Ciudad Real)
* **Year:** 2026

---

## 2. INTRODUCTION AND STRATEGY

We built our robot using the components provided to our school through the **ESCUELA 4.0** program. Combining this material with a 3D printer and a laser cutter, we designed and manufactured all the necessary structural parts.

### 2.1 Component and Software Summary
Our robot is built around a **BBC micro:bit V2** coupled with a **KeyStudio expansion shield**. Connected to this setup are:
* A **Huskylens AI camera** for vision tracking.
* An **SG90 180° servo motor** to control a Lego-built rack-and-pinion steering mechanism.
* A geared **DC motor** driving the rear axle for propulsion.
* Three **HC-SR04 ultrasonic distance sensors** to avoid wall collisions.
* An **L298N motor driver module**.
* A battery holder for two **18650 Li-ion batteries** and a power switch.

The robot is capable of navigating an enclosed circuit between walls, avoiding obstacles, and steering to the right when it detects red prisms, or to the left when it detects green prisms.

### 2.2 Objectives
1. Design a lightweight robot using the available school components.
2. Achieve consistent and reliable baseline movement.
3. Complete full laps around the track autonomously.
4. Correctly identify and avoid colored game pieces by steering to the appropriate side.

---

## 3. MECHANICAL DESIGN (HARDWARE)

### 3.1 Chassis Structure
The chassis is made from **3mm MDF (Medium-Density Fibreboard)**, designed in *Inkscape* and manufactured using a laser cutter. It features a dual-layer layout:
* **Lower Deck:** Houses the drive motor, steering assembly, batteries, and servo motor to maintain a low center of gravity.
* **Upper Deck:** Hosts the micro:bit, expansion shield, motor controller, and sensors.

Additional custom components were 3D-printed, including custom gears, camera mounts, axles, and ultrasonic sensor mounts, assembled using standard screws and expansion kit pieces.

* **Total Weight:** 1.2 kg
* **Dimensions:** 120 x 180 x 150 mm

### 3.2 Steering System
The vehicle implements an **Ackermann-inspired steering geometry**, executed mechanically using a Lego rack-and-pinion mechanism. This assembly is driven by an SG90 servo motor linked via custom 3D-printed gears.

### 3.3 Drivetrain & Traction
* **Motor:** Geared DC motor with a 1:48 reduction ratio.
* **Gear Ratio:** 1:1 (using two identical interlocking gears).
* **Drive Type:** Rear-wheel drive (RWD).

### 3.4 Component Distribution
By placing the heavy batteries and the drive motor on the lowest deck, we effectively minimized the center of mass to maximize stability. The upper section houses the control electronics and vision systems to keep them protected and clear of mechanical linkages.

---

## 4. ELECTRONICS AND SENSORS

### 4.1 Main Controller
* **BBC micro:bit V2:** Selected due to its accessibility, block-based programming environment, and direct availability at our school. It handles all control logic via its I2C and digital interfaces.

### 4.2 Vision Sensors
* **Huskylens V1 AI Camera:** Embedded vision processor fully compatible with the micro:bit via I2C, featuring native algorithms for color and object recognition.

### 4.3 Distance and Orientation Sensors
* **Distance Sensors:** 3x **HC-SR04 ultrasonic sensors** configured to measure distances to front, left, and right walls to prevent collisions.
* **Gyroscope:** While the micro:bit features a built-in internal gyroscope and compass, we encountered programming limitations and did not successfully integrate them into the final steering loop.

### 4.4 Batteries & Power Management
* **Cells:** 2x **Panasonic NCR18650B Li-ion cells** (3400 mAh, 3.7V each, wired in series).
* **Regulation:** Voltage regulation is managed directly through the KeyStudio sensor expansion shield to distribute clean power across all modules.

---

## 5. SOFTWARE (PROGRAMMING)

### 5.1 Development Environment
The software was developed using the cloud-based **Microsoft MakeCode** platform utilizing standard extension blocks rather than low-level text libraries, maintaining a Scratch-like block architecture.

### 5.2 AI Camera Setup
Using the Huskylens extension, the initialization routine starts the camera over an I2C connection and switches the internal algorithm to color recognition mode.

### 5.3 Flowchart Logic
The robot continuously reads distance data while looking out for any visual color triggers. It branches its navigation rules based on whether a colored target block is active within the camera's view matrix.

### 5.4 Source Code (MakeCode JavaScript)

```javascript
// Function to move backward
function Atras (tiempo: number) {
    pins.digitalWritePin(DigitalPin.P8, 0)
    pins.digitalWritePin(DigitalPin.P2, 1)
    pins.analogWritePin(AnalogPin.P1, 600)
    basic.pause(tiempo)
    pins.digitalWritePin(DigitalPin.P1, 0)
}

// Function to move forward
function Adelante (tiempo: number) {
    pins.digitalWritePin(DigitalPin.P2, 0)
    pins.digitalWritePin(DigitalPin.P8, 1)
    pins.analogWritePin(AnalogPin.P1, 600)
    basic.pause(tiempo)
    pins.digitalWritePin(DigitalPin.P1, 0)
}

// Button A event trigger: Starts loops for round 1&2 (no objects) or 3&4 (with objects)
input.onButtonPressed(Button.A, function () {
    c = huskylens.readBox_s(Content3.ID)
    if (c <= 0) {
        while (true) {
            sinobjetos()
        }
    } else {
        while (true) {
            objetos()
        }
    }
})

// Navigation loop for rounds 3 and 4 (Obstacle interaction)
function objetos () {
    // Define and fetch data from variables
    c = huskylens.readBox_s(Content3.ID)
    posx = huskylens.readeBox_index(1, 1, Content1.xCenter)
    huskylens.writeName(1, "Rojo")
    
    df = sonar.ping(
    DigitalPin.P16,
    DigitalPin.P15,
    PingUnit.Centimeters
    )
    dd = sonar.ping(
    DigitalPin.P13,
    DigitalPin.P14,
    PingUnit.Centimeters
    )
    di = sonar.ping(
    DigitalPin.P12,
    DigitalPin.P11,
    PingUnit.Centimeters
    )
    
    if (c <= 0) {
        // When approaching a front wall
        if (df < 26) {
            // Evaluate best opening to turn
            if (dd > di) {
                basic.showIcon(IconNames.House)
                parar(100)
                servos.P0.setAngle(55)
                Atras(600)
                parar(100)
                servos.P0.setAngle(125)
                Adelante(300)
            } else {
                basic.showIcon(IconNames.Cow)
                parar(100)
                servos.P0.setAngle(125)
                Atras(600)
                parar(100)
                servos.P0.setAngle(55)
                Adelante(300)
            }
        } else {
            // Keep centered between side walls
            if (dd > di || 10 > di) {
                parar(50)
                servos.P0.setAngle(125)
                Adelante(200)
            } else if (dd < di || 10 > dd) {
                parar(50)
                servos.P0.setAngle(55)
                Adelante(200)
            }
        }
    } else if (c == 1) {
        color_rojo()
    } else if (c == 2) {
        color_verde()
    }
}

// Green object handling routine
function color_verde () {
    if (posx >= 160) {
        servos.P0.setAngle(110)
        Atras(500)
        servos.P0.setAngle(55)
        Adelante(400)
    } else {
        servos.P0.setAngle(110)
        Atras(500)
        servos.P0.setAngle(55)
        Adelante(400)
    }
}

// Base navigation loop when no objects are detected (Rounds 1 & 2)
function sinobjetos () {
    df = sonar.ping(
    DigitalPin.P16,
    DigitalPin.P15,
    PingUnit.Centimeters
    )
    dd = sonar.ping(
    DigitalPin.P13,
    DigitalPin.P14,
    PingUnit.Centimeters
    )
    di = sonar.ping(
    DigitalPin.P12,
    DigitalPin.P11,
    PingUnit.Centimeters
    )
    
    if (df < 25) {
        if (dd > di) {
            basic.showIcon(IconNames.House)
            parar(100)
            servos.P0.setAngle(55)
            Atras(600)
            parar(100)
            servos.P0.setAngle(110)
            Adelante(300)
        } else {
            basic.showIcon(IconNames.Cow)
            parar(100)
            servos.P0.setAngle(110)
            Atras(600)
            parar(100)
            servos.P0.setAngle(55)
            Adelante(300)
        }
    } else {
        if (dd > di || 10 > di) {
            servos.P0.setAngle(110)
        } else if (dd < di || 10 > dd) {
            servos.P0.setAngle(55)
        }
        Adelante(100)
    }
}

// Red object handling routine
function color_rojo () {
    if (posx <= 160) {
        servos.P0.setAngle(55)
        Atras(500)
        servos.P0.setAngle(110)
        Adelante(400)
    } else {
        servos.P0.setAngle(55)
        Atras(500)
        servos.P0.setAngle(110)
        Adelante(400)
    }
}

// Braking and delay function
function parar (t: number) {
    pins.analogWritePin(AnalogPin.P1, 0)
    basic.pause(t)
}

// Global variable initializations
let di = 0
let dd = 0
let df = 0
let posx = 0
let c = 0
servos.P0.setAngle(90)
c = 0
huskylens.initI2c()
huskylens.initMode(protocolAlgorithm.ALGORITHM_COLOR_RECOGNITION)
pins.digitalWritePin(DigitalPin.P2, 0)
pins.digitalWritePin(DigitalPin.P8, 1)

loops.everyInterval(500, function () {
    huskylens.request()
})
