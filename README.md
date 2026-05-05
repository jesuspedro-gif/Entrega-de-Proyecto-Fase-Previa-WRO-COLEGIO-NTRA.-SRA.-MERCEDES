WRO ROBOTICS SYMPOSIUM, MAY 2026 – CIUDAD REALCOLEGIO NUESTRA SEÑORA DE LAS MERCEDES –HERENCIA
-Technical DescriptionDevelopment: 
The robot's development began with a small wooden board used for prototyping. We then designed the chassis using vector drawing software to manufacture it with a laser cutter. 
The structure consists of two levels: the lower level houses the steering system, servo motor, and batteries, while the upper level contains the electronic components.
Components:DC motor with gearbox.L298N motor driver.Micro:bit board + Keyestudio Shield.18650 LiPo battery holder (2 in series).HuskyLens V1 AI Camera.Power switch.180º Servo motor.Lego wheels.Rack and pinion + linkage system (built with Lego parts).3D printed parts: gears, mounts, and couplings.Challenges: After resolving a chassis issue, we designed the upper deck to hold the Micro:bit, its expansion board, the power switch, and the L298N. We also integrated a mount to secure the 180º servo on the lower section. 
During assembly, we installed the battery pack and the geared DC motor. Finally, we added the pillars to join both levels and wired all components. A hardware setback occurred when the switch broke; it was replaced with the current one. Lastly, we 3D printed a custom mount to elevate the HuskyLens AI camera above other components and connected it to the expansion board. Once the hardware was finalized, we began programming and testing on custom-made obstacle courses to debug the code.Technical DecisionsAI Camera 
Implementation: Elevated by several centimeters to improve the field of view (horizon control).Core Hardware: Use of Micro:bit and dedicated expansion shields.
Form Factor: Compact robot design.ResultsWe successfully built a functional basic robot. Although its movement did not meet our initial expectations, we believe that with more time and dedication, it could have performed at a high level.
Mobility:
180º Servo for steering (angles: ±45º).
Geared DC motor for rear-wheel drive.Lego-based wheels and chassis components.L298N driver for DC motor power management.
Power Supply:Two 18650 LiPo batteries.Sensors:HuskyLens AI Camera.
Code Documentation (Excerpts)typescript/**


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
Operation
The objective is for the robot to autonomously avoid obstacles based on specific instructions. To achieve this, the system was programmed with two main goals:Obstacle Avoidance: Represented by colors (IDs) 1, 2, and 3.Parking (Goal): Represented by color (ID) 4 (Note: Goal not fully achieved).
To accomplish this, the code integrates a DC motor for traction, a servo motor for steering, and a HuskyLens camera for object (color) recognition.
Startup and MechanicsSteering: The robot utilizes a "rack and pinion" steering system controlled by a servo motor connected to pin P0. 
A 90-degree angle represents the neutral position (straight wheels), while angles 55 and 135 are used for lateral turns.
Traction (DC Motor on P1, P2, and P8): Forward and backward movement is managed via three pins. P2 and P8 act as logic switches (0 or 1) to determine motor direction. Pin P1 sends a PWM (analog) signal to regulate speed, ranging from 0 (stopped) to 1023 (max speed).
Camera (HuskyLens): I2C communication is initialized, and the camera is configured in "Color Recognition" mode.Movement FunctionsWe developed auxiliary functions to streamline the logic:Forward (speed, time) / Backward (speed, time): Moves the robot for a specific duration in milliseconds.Stop: Immediately cuts power to the DC motor.
Main LoopUpon pressing Button A, the robot enters an infinite loop:Status Check: It first checks if the course is finished; if so, it stops all motors and exits the loop.
Analysis: If active, the HuskyLens analyzes the field of view and retrieves the ID (color) of the object detected in the center of the frame. Based on this, the robot makes one of three decisions: park, avoid an obstacle, or move forward.Logic ScenariosParking Mode (ID 4): Once detected, modoMeta is set to true, giving parking absolute priority.Acceleration: The robot moves at maximum speed (1023).Coordinates: Using X (horizontal) and Y (vertical) coordinates, the robot adjusts:X < 140: Servo turns to angle 55.X > 180: Servo turns to angle 135.140 < X < 180: Wheels straighten (90°).Proximity: In ground robotics computer vision, as an object "descends" in the frame (Y increases), it means the robot is getting closer. If Y > 200, the robot assumes it has reached the goal, sets carreraTerminada to true, stops, and displays an "X" (IconNames.No) on the LED matrix.Wall Detection (ID 3): The robot brakes, steers away from the wall's center, reverses, and performs an "S" maneuver forward to bypass it.Small Obstacles (IDs 1 & 2): The robot turns the wheels, reverses slightly to create space, and then straightens up. The code assigns specific turn directions based on whether the color is Red (1) or Green (2).Default State: When no objects are detected, the steering stays straight (90°) and the robot moves at a moderate speed of 750, constantly scanning its surroundings.
Demonstration Video:Watch the robot operating autonomously across different sections here:https://youtube.com/shorts/AhCgXqGC2gI?feature=share
