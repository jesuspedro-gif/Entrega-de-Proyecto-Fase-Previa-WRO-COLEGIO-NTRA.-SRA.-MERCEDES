# WRO Robotics Symposium, May 2026 – Ciudad Real
**Colegio Nuestra Señora de las Mercedes – Herencia**

---

## Technical Description

### Development
The robot's development began with a small wooden board used for prototyping. We then designed the chassis using vector drawing software to manufacture it with a laser cutter. 

The structure consists of two distinct levels:
* **Lower Level:** Houses the steering system, servo motor, and batteries.
* **Upper Level:** Contains the electronic components.

### Challenges
After resolving a chassis issue, we designed the upper deck to hold the Micro:bit, its expansion board, the power switch, and the L298N driver. We also integrated a mount to secure the 180º servo on the lower section. 

During assembly, we installed the battery pack and the geared DC motor. We added pillars to join both levels and wired all components. A hardware setback occurred when the switch broke; it was replaced with the current, more durable one. Lastly, we 3D printed a custom mount to elevate the HuskyLens AI camera above other components and connected it to the expansion board. Once the hardware was finalized, we began programming and testing on custom-made obstacle courses to debug the code.

---

## Hardware Components

| Category | Components |
| :--- | :--- |
| **Microcontroller** | Micro:bit board, Keyestudio Shield |
| **Motors & Drivers** | DC motor with gearbox, L298N motor driver, 180º Servo motor |
| **Sensors / Vision** | HuskyLens V1 AI Camera |
| **Power Supply** | 18650 LiPo battery holder (2 in series), Power switch |
| **Mechanics** | Lego wheels, Rack and pinion + linkage system (Lego parts) |
| **Custom Elements** | 3D printed parts (gears, mounts, and couplings) |

---

## Technical Decisions & Results

We successfully built a functional basic robot. Although its movement did not meet our initial expectations, we believe that with more time and dedication, it could have performed at a high level.

### Design Choices
* **AI Camera Implementation:** Elevated by several centimeters to improve the field of view and maintain horizon control.
* **Core Hardware:** Relied on Micro:bit and dedicated expansion shields for accessible and robust processing.
* **Form Factor:** Maintained a compact robot design to easily navigate tight courses.

### Mobility & Power
* **Steering:** Managed by a 180º Servo (operational angles: ±45º).
* **Traction:** Geared DC motor providing rear-wheel drive.
* **Power Management:** L298N driver dedicated to DC motor power distribution.

---

## Operation & Logic

The objective is for the robot to autonomously avoid obstacles based on specific environmental instructions. The system was programmed with two main goals:
1.  **Obstacle Avoidance:** Represented by object colors (IDs) 1, 2, and 3.
2.  **Parking (Goal):** Represented by color (ID) 4 *(Note: Goal not fully achieved during final testing)*.

### Startup and Mechanics
* **Steering Logic:** The robot utilizes a "rack and pinion" steering system controlled by a servo motor on pin `P0`. A 90° angle represents the neutral position (straight wheels), while 55° and 135° trigger lateral turns.
* **Traction Logic:** Forward and backward movement is managed via three pins (`P1`, `P2`, `P8`). `P2` and `P8` act as logic switches (0 or 1) to determine direction. `P1` sends a PWM (analog) signal to regulate speed, ranging from 0 (stopped) to 1023 (max speed).
* **Camera Logic:** The HuskyLens is initialized via I2C communication and configured exclusively in "Color Recognition" mode.

### Behavioral Scenarios
* **Parking Mode (ID 4):** Once detected, parking takes absolute priority. The robot accelerates to maximum speed (1023) and dynamically adjusts steering to keep the target centered. If the object's Y-coordinate exceeds 200 (meaning it is directly in front/below), the robot assumes it has reached the goal, stops, and displays an "X" on the LED matrix.
* **Wall Detection (ID 3):** The robot immediately brakes, steers away from the wall's center, reverses, and executes an "S" maneuver to safely bypass it.
* **Small Obstacles (IDs 1 & 2):** The robot turns its wheels, reverses slightly to create space, and straightens up. Turn directions are dictated by whether the detected color is Red (1) or Green (2).
* **Default State:** When no objects are detected, the steering remains straight (90°) and the robot moves at a moderate speed of 750, constantly scanning for new inputs.

---

## Code Documentation (Excerpts)

```typescript
/**
 * Continuous forward motion function.
 * Sets traction pins and applies speed.
 */
function moveForward(speed: number) {
    // P2 to 0 and P8 to 1 set the motor direction (Forward)
    pins.digitalWritePin(DigitalPin.P2, 0)
    pins.digitalWritePin(DigitalPin.P8, 1)
    // P1 regulates speed via analog signal (0 to 1023)
    pins.analogWritePin(AnalogPin.P1, speed)
}

// MAIN LOOP
input.onButtonPressed(Button.A, function () {
    while (true) {
        // 1. Exit condition: Stop if the race is finished.
        if (raceFinished) {
            stop(0)
            return
        }

        // 2. HuskyLens Data Request
        huskylens.request()
        Color = huskylens.readBox_s(Content3.ID)

        // If ID 4 (Finish Line) is detected, activate Finish Mode
        if (Color == 4) {
            finishMode = true
        }

        // ==========================================
        // FINISH LINE LOGIC (Targeting Color 4)
        // ==========================================
        if (finishMode) {
            basic.showIcon(IconNames.Target)
            if (Color == 4) {
                posX = huskylens.readeBox(Color, Content1.xCenter)
                posY = huskylens.readeBox(Color, Content1.yCenter)

                // Steering Logic: Keep the object centered
                if (posX < 140) {
                    servos.P0.setAngle(55) // Correct left
                } else if (posX > 180) {
                    servos.P0.setAngle(135) // Correct right
                } else {
                    servos.P0.setAngle(90) // Straight
                }

                moveForward(1023) // Max speed towards finish

                // If posY > 200, the object is at the bottom (directly in front)
                if (posY > 200) {
                    raceFinished = true
                    stop(0)
                    basic.showIcon(IconNames.No)
                }
            } else {
                // If finish line is lost, try going straight for 0.5s
                servos.P0.setAngle(90)
                moveForward(1023)
                basic.pause(500)
                raceFinished = true
                stop(0)
                basic.showIcon(IconNames.No)
            }
            return
        }

        // ==========================================
        // OBSTACLE AVOIDANCE LOGIC (IDs 1, 2, and 3)
        // ==========================================
        if (Color > 0 && Color <= 3) {
            posX = huskylens.readeBox(Color, Content1.xCenter)
            posY = huskylens.readeBox(Color, Content1.yCenter)
            width = huskylens.readeBox(Color, Content1.width)
            height = huskylens.readeBox(Color, Content1.height)

            // --- COLLISION DETECTION ---
            // Reverse if: object is low (near), too wide (wall), or too tall.
            if (posY > 140 || width > 100 || height > 100) {
                if (Color == 3) {
                    // WALL LOGIC (Color 3): Reverse while steering away
                    if (posX >= 160) {
                        servos.P0.setAngle(135)
                    } else {
                        servos.P0.setAngle(55)
                    }
                    stop(150)
                    reverse(1023, 1000)
                    stop(150)
                    
                    // Exit maneuver (S-curve)
                    if (posX >= 160) {
                        servos.P0.setAngle(55)
                    } else {
                        servos.P0.setAngle(135)
                    }
                    moveForwardTimed(750, 600)
                    
                    if (posX >= 160) {
                        servos.P0.setAngle(135)
                    } else {
                        servos.P0.setAngle(55)
                    }
                    moveForwardTimed(750, 400)
                } else {
                    // SMALL OBSTACLES (IDs 1 & 2): Quick reverse tap
                    if (Color == 1) {
                        servos.P0.setAngle(55)
                    } else {
                        servos.P0.setAngle(135)
                    }
                    stop(150)
                    reverse(1023, 400)
                }
                servos.P0.setAngle(90) // Straighten wheels
            }
        }
    }
})
```

---

## Demonstration

Watch the robot operating autonomously across different sections here:  
[**Autonomous Operation Video (YouTube Shorts)**](https://youtube.com/shorts/AhCgXqGC2gI?feature=share)
