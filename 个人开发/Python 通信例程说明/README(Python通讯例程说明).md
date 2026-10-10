# Unitree J288 / S288 Digital Servo Python Serial Communication Demo
> Official Python serial communication demo for Unitree J288/S288 brushless joint servo, based on pyserial library, compatible with Windows/Linux/macOS, support send control command & receive servo feedback data.

Product Official Page: https://www.unitree.com/en/DigitalServo

## 1. Environment Preparation
### 1.1 Dependency Installation
This demo relies on `pyserial` serial port library, execute the pip command to install:
```bash
pip install pyserial
```

### 1.2 Serial Port Configuration
1. Confirm your serial port device name:
   - Windows: `COM3` / `COM4` etc. (Check via Device Manager)
   - Linux: `/dev/ttyUSB0` / `/dev/ttyACM0`
   - macOS: `/dev/tty.usbserial-xxxx`
2. Modify the serial port parameter inside the python source code to match your actual device
3. Serial communication fixed parameters:
   - Baudrate: 6000000 (6Mbps)
   - Frame format: 8N1 (8 data bit, 1 stop bit, no parity)
   - Communication mode: Half-duplex TTL bus

## 2. Run Demo
1. Complete wiring between PC serial adapter and J288/S288 servo, power on the servo
2. Modify serial port string in code
3. Execute the script:
```bash
python servo_demo.py
```
4. Follow the console prompt to input control parameters and send commands to the servo

## 3. Communication Logic Description
1. After sending control command frame, servo will immediately return real-time feedback data once communication completed
2. `timeout` parameter logic:
   - `timeout = 1`: Enable timeout protection. Servo only execute command once, you need to continuously send control frames to keep motion
   - `timeout = 0`: Disable timeout protection. Single command send, servo will keep running according to this parameter persistently

## 4. All Control Parameter Definition
| Parameter | Value Range | Function Description |
|-----------|-------------|----------------------|
| id | 0 ~ 15 | Unique bus ID of target servo, single bus supports max 16 servos |
| mode | 0 / 1 | Servo running mode: `0` stop & lock torque; `1` hybrid closed-loop control (speed/position/torque) |
| timeout | 0 / 1 | Timeout protection switch |
| outputTor | -36.909 ~ 36.908 | Target output shaft torque (Unit: N·m) |
| outputSpd | -278.902 ~ 278.902 | Target output shaft speed (Unit: rad/s) |
| outputPos | -1428.019 ~ 1428.019 | Target output shaft angle (Unit: rad) |
| Kp | 0 ~ 2128.523 | Output stiffness proportional gain |
| Kd | 0 ~ 21.285 | Output damping differential gain |

## 5. Basic Operation Example
### 5.1 Constant Speed Rotation
Set parameters as below:
- `id = 0` (Target servo ID 0)
- `mode = 1` (Enable closed-loop motion)
- `timeout = 0` (Turn off timeout protection, keep running)
- `outputSpd = 0.5` (Fixed speed 0.5 rad/s)
- `Kd = 1` (Set damping coefficient)
After sending command, servo will rotate continuously at fixed speed.

### 5.2 Stop Servo
Set `mode = 0` and send command, servo stop motion and lock output torque.

## 6. Troubleshooting
1. Cannot open serial port
   - Linux/macOS: Add serial port permission `sudo chmod 666 /dev/ttyUSB0`
   - Windows: Confirm serial port number correct, no other software occupy the port
2. No feedback data from servo
   - Check wiring TX/RX direction, confirm GND common ground
   - Verify baud rate is set to 6000000, not default 115200
   - Check servo power supply matches model voltage requirement
3. Servo stop running after a short while
   - Check if `timeout=1` is enabled, need cyclic send command frame

## 7. License
This python demo is open for non-commercial research, education and DIY development.
For commercial mass production application, please contact Unitree official for formal authorization.

## Official Product Link
https://www.unitree.com/en/DigitalServo