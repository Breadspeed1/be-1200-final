# ESP32 Security System Tutorial
*Arduino-based door alarm with RFID arming, reed switch trigger, and ESP32-CAM photo capture*

## Project Overview

This project builds a simple security system using two ESP32 boards:

- **Main ESP32 Dev Board**: Reads an RFID card for arming/disarming, monitors a magnetic reed switch on a door, controls status LEDs, and coordinates with the camera board.
- **ESP32-CAM Board**: Listens for commands over UART. When it receives "PHOTO", it takes three pictures, saves them to a microSD card, and sends the last image back to the main board.

**How it works**:
1. User taps an authorized RFID card to arm the system (green LED on).
2. If the door opens (reed switch triggers) while armed, the main board sends a "PHOTO" command via UART to the CAM board.
3. The CAM board captures three photos, saves all to the SD card, and streams the last photo back to the main board.
4. User taps the RFID card again to disarm (red LED on).

**Key concepts for students**: UART communication, SPI for RFID, GPIO input/output, task scheduling on dual-core ESP32, SD card file writing.

---

## Bill of Materials

| Component | Quantity | Notes | Example Purchase Link |
|-----------|----------|-------|----------------------|
| ESP32 Dev Board (Dev Module) | 1 | Main controller, has USB port | [Amazon](https://www.amazon.com/dp/B0D8T53CQ5) |
| AI Thinker ESP32-CAM | 1 | Camera module with microSD slot, no USB port | [Amazon](https://www.amazon.com/dp/B09TB1GJ7P) |
| CP2102 USB-to-TTL Adapter | 1 | For flashing the ESP32-CAM | [Amazon](https://www.amazon.com/dp/B00LODGRV8) |
| MFRC522 RFID Module | 1 | Includes RFID card/tag | (add your link) |
| Magnetic Reed Switch | 1 | Normally open type | [Amazon](https://www.amazon.com/gp/product/B07S58HH3Q) |
| MicroSD Card | 1 | 32GB, FAT32 formatted | [Amazon](https://www.amazon.com/dp/B0749KG1JK) |
| LEDs (Red, Green) | 1 each | 5mm or 3mm | (add your link) |
| Resistors | 2x 220 ohm | For LED current limiting | (add your link) |
| Breadboard + Jumper Wires | 1 set | Dual power rails recommended | (add your link) |
| External Power Supply | 1 | Provides both 5V and 3.3V rails | (specify yours) |

**Power note**: All logic signals (GPIO pins) on ESP32 boards operate at 3.3V. The RC522 RFID module and reed switch in this project are powered at 3.3V. The ESP32-CAM is powered at 5V via its 5V pin, but its GPIO pins remain 3.3V logic.

---

## Wiring and Assembly

[INSERT YOUR TINKERCAD DIAGRAM HERE: Full system overview]

### Power Rail Setup

1. Set up your breadboard with two separate power rails:
   - **5V rail**: Connect to the ESP32-CAM 5V pin and to the anodes of the LEDs (through 220 ohm resistors).
   - **3.3V rail**: Connect to the ESP32 Dev Board 3.3V pin, the RC522 VCC pin, and one side of the reed switch.
2. Connect all ground (GND) pins together: ESP32 Dev Board GND, ESP32-CAM GND, RC522 GND, reed switch GND side, and power supply GND. This common ground is essential for communication.
3. **Important when flashing the main ESP32 via USB**: Temporarily disconnect the 5V VIN wire that runs from the external power supply to the main ESP32 breadboard connection. Keep the GND connection intact. This prevents conflict between USB power and the external supply. Reconnect the VIN wire after uploading code.

### Pin Connections

[INSERT CLOSE-UP DIAGRAMS HERE for: Main ESP32 connections, ESP32-CAM connections, UART cross-connection]

#### Main ESP32 Dev Board Connections

| Main ESP32 Pin | Connects To | Notes |
|----------------|-------------|-------|
| 3.3V | RC522 VCC, one side of reed switch | All logic at 3.3V |
| GND | RC522 GND, reed switch other side, ESP32-CAM GND, LED cathodes | Common ground |
| GPIO 14 | Reed switch signal side | Configured as INPUT in code |
| GPIO 19 | Green LED anode (via 220 ohm resistor) | Cathode to GND |
| GPIO 18 | Red LED anode (via 220 ohm resistor) | Cathode to GND |
| GPIO 17 (TX2) | ESP32-CAM GPIO 3 (U0RXD) | UART2 transmit to CAM receive |
| GPIO 16 (RX2) | ESP32-CAM GPIO 1 (U0TXD) | UART2 receive from CAM transmit |
| GPIO 26 | RC522 SDA (MOSI) | SPI data |
| GPIO 32 | RC522 SCK | SPI clock |
| GPIO 33 | RC522 MOSI | SPI master out |
| GPIO 25 | RC522 MISO | SPI master in |
| GPIO 27 | RC522 RST | Reset pin |

#### ESP32-CAM UART Pin Reference

The AI Thinker ESP32-CAM has two pins labeled **U0RXD** and **U0TXD** located just above the GND pin on the board edge. These correspond to:
- **U0RXD = GPIO 3** (receive pin for UART0)
- **U0TXD = GPIO 1** (transmit pin for UART0)

Connect these to the main board as shown in the table above. Double-check this cross-connection: TX on one board must go to RX on the other.

#### ESP32-CAM Flashing Setup (CP2102 Adapter)

[INSERT DIAGRAM: CP2102 to ESP32-CAM wiring]

| CP2102 Pin | ESP32-CAM Pin | Purpose |
|------------|---------------|---------|
| 3.3V | 3.3V | Power during flash only (do not use 5V) |
| GND | GND | Common ground |
| TX | GPIO 1 (U0TXD) | Data to CAM receive pin |
| RX | GPIO 3 (U0RXD) | Data from CAM transmit pin |
| GND | GPIO 0 | Hold LOW to enter flash mode |

**Flashing steps**:
1. Wire the CP2102 adapter as shown above, ensuring GPIO 0 is connected to GND.
2. In Arduino IDE, select Board: "AI Thinker ESP32-CAM".
3. Upload the `cam_board.cpp` sketch.
4. After upload completes, remove the wire from GPIO 0, then press the ESP32-CAM's EN/RST button to restart.
5. Open Serial Monitor at 115200 baud. You should see "CAM_READY".

For detailed visual instructions, follow the guide you referenced:  
[Random Nerd Tutorials: Program ESP32-CAM](https://randomnerdtutorials.com/program-upload-code-esp32-cam/)

#### SD Card Preparation

- Format your 32GB microSD card as FAT32 using the [SD Memory Card Formatter](https://www.sdcard.org/downloads/formatter/).
- Insert the card into the ESP32-CAM slot before powering on the board.
- Photos will be saved with filenames like `/photo_1.jpg`, `/photo_2.jpg`, etc.

---

## Software Setup

### Step 1: Install Arduino IDE and ESP32 Board Support

1. Download and install the Arduino IDE from https://www.arduino.cc/en/software
2. Open the Arduino IDE.
3. Go to File > Preferences.
4. In the "Additional Boards Manager URLs" field, paste:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
5. Go to Tools > Board > Boards Manager.
6. Search for "esp32" and install "ESP32 by Espressif Systems" (version 2.0.17 or later).
7. Restart the Arduino IDE.

### Step 2: Install Required Libraries

1. In Arduino IDE, go to Tools > Manage Libraries.
2. Search for and install:
   - `MFRC522` by Miguel Balboa (version 1.4.12 or later)
3. Note: The `esp_camera` library used by the CAM board is built into the ESP32 board package and does not need separate installation.

### Step 3: Prepare the Code Files

Save the provided code files as separate `.ino` files in the same project folder:
- `main_board.ino` (for the main ESP32 Dev Board)
- `cam_board.ino` (for the ESP32-CAM)

#### Configuring main_board.ino

1. Open `main_board.ino` in Arduino IDE.
2. Find your RFID card's UID:
   - Upload the code temporarily without changes.
   - Open Serial Monitor at 115200 baud.
   - Tap your authorized RFID card near the reader.
   - Note the printed UID (e.g., `F9 49 B0 05`).
   - Stop the upload, then edit line 25 to match your card:
     ```cpp
     byte authorizedUID[4] = {0xF9, 0x49, 0xB0, 0x05}; // Replace with YOUR UID bytes
     ```
3. Verify the pin definitions match your wiring (lines 10-17):
   ```cpp
   #define REED_PIN 14
   #define CAM_TX 17   // Main TX -> CAM RX (GPIO 3)
   #define CAM_RX 16   // Main RX <- CAM TX (GPIO 1)
   #define RC_SDA 26   // SPI pins for RC522
   #define RC_SCK 32
   #define RC_MOS 33
   #define RC_IMI 25
   #define RC_RST 27
   ```

#### Configuring cam_board.ino

1. Open `cam_board.ino` in Arduino IDE.
2. The camera configuration (lines 13-35) is pre-set for the AI Thinker ESP32-CAM. No changes are needed for standard hardware.
3. Optional: Adjust photo settings if desired:
   ```cpp
   config.frame_size = FRAMESIZE_VGA;  // Options: FRAMESIZE_CIF, VGA, SVGA, etc.
   config.jpeg_quality = 10;           // Lower number = better quality, larger file size
   ```

### Step 4: Upload to Main ESP32 Dev Board

1. Connect the main ESP32 Dev Board to your computer via USB cable.
2. In Arduino IDE, configure:
   - Board: "ESP32 Dev Module"
   - Port: [Select your COM port]
   - Upload Speed: 921600 (faster) or 115200 (more reliable)
3. **Critical**: Disconnect the 5V VIN wire from the breadboard connection to the main ESP32 (keep GND connected).
4. Click the Upload button.
5. After upload completes, reconnect the VIN wire to restore full power from the external supply.

### Step 5: Upload to ESP32-CAM Using CP2102 Adapter

1. Wire the CP2102 adapter to the ESP32-CAM as described in the Wiring section.
2. In Arduino IDE, configure:
   - Board: "AI Thinker ESP32-CAM"
   - Port: [Select the CP2102 COM port]
   - Ensure GPIO 0 is grounded during upload
3. Click Upload.
4. After upload:
   - Remove the wire connecting GPIO 0 to GND.
   - Press the ESP32-CAM's EN/RST button to restart.
   - Open Serial Monitor at 115200 baud. You should see "CAM_READY".

### Step 6: Final System Check

1. Power both boards using the external power supply (ensure all wires are reconnected).
2. Open Serial Monitor for the main board at 115200 baud.
3. Expected startup output:
   ```
   ====================================
      Anti-Theft System
      Main ESP32
   ====================================
   Image buffer allocated (50 KB)
   RFID scanner initialized.
   UART2  ->  ESP32-CAM
   Waiting for peripherals...
     CAM: CAM_READY
   System ready - DISARMED
   ```
4. If you see "System ready - DISARMED", both boards are communicating correctly.

---

## Testing and Operation

1. **Arm the system**: Tap the authorized RFID card near the reader. The green LED should turn on, and the Serial Monitor should display "System armed via RFID".
2. **Trigger the alarm**: Open the door (separate the reed switch contacts). Observe:
   - Serial Monitor shows: `*** ALARM TRIGGERED: Door Opened ***`
   - The ESP32-CAM saves three photos to the SD card (`/photo_1.jpg`, `/photo_2.jpg`, `/photo_3.jpg`)
   - The last photo is streamed back to the main board and buffered in memory
3. **Disarm the system**: Tap the authorized RFID card again. The red LED should turn on, and the Serial Monitor should display "System disarmed via RFID".
4. **View captured photos**: Power down the system, remove the microSD card from the ESP32-CAM, and insert it into a computer. Open the `photo_*.jpg` files with any image viewer.

---

## Troubleshooting

### CAM board not responding or "CAM did not report ready"

- **Check UART wiring**: Verify that Main GPIO 17 connects to CAM GPIO 3 (U0RXD), and Main GPIO 16 connects to CAM GPIO 1 (U0TXD). TX must go to RX.
- **Check common ground**: Ensure all GND pins are connected together.
- **Check CAM power**: The ESP32-CAM 5V pin must be connected to the 5V rail. During normal operation, GPIO 0 should not be grounded.
- **Check Serial Monitor baud rate**: Both boards use 115200 baud. Ensure your monitor is set correctly.
- **Re-flash the CAM board**: Follow the flashing steps again, ensuring GPIO 0 is grounded only during upload.

### RFID card not detected

- **Check SPI wiring**: Verify connections for SDA, SCK, MOSI, MISO, RST, and GND between the main ESP32 and RC522.
- **Check RC522 power**: The module VCC should be connected to 3.3V (not 5V) as confirmed in your build.
- **Check antenna orientation**: Hold the RFID card flat against the reader module for best results.
- **Verify UID in code**: Ensure the `authorizedUID` array in `main_board.ino` exactly matches your card's printed UID bytes.

### SD card write errors on ESP32-CAM

- **Format as FAT32**: Use the official SD Memory Card Formatter tool. Do not use exFAT or NTFS.
- **Check card insertion**: Ensure the microSD card is fully seated in the slot.
- **Check card capacity**: The project was tested with a 32GB card. Larger cards may require additional configuration.
- **Power stability**: The ESP32-CAM draws significant current during photo capture. Ensure your 5V supply can provide at least 500mA.

### Photos not streaming back to main board

- **Check UART buffer settings**: The main board code sets `Serial2.setRxBufferSize(2048)`. Do not change this unless you also adjust the receive logic.
- **Check for interference**: Keep UART wires short and away from high-current paths.
- **Verify protocol timing**: The CAM board sends "IMG:<size>" followed by raw bytes and "IMG_END". Ensure no extra serial output is interfering.

### LEDs not lighting correctly

- **Check resistor orientation**: LEDs are polarized. The longer leg (anode) connects to the GPIO pin through the resistor; the shorter leg (cathode) connects to GND.
- **Check GPIO definitions**: Verify that `LED_GREEN` and `LED_RED` in the code match your physical wiring (GPIO 19 and 18 by default).
- **Check power rail**: LED anodes should connect to the 5V rail through resistors; cathodes to GND.

### General upload issues

- **ESP32 Dev Board not detected**: Try a different USB cable. Some cables are power-only.
- **ESP32-CAM upload fails**: Ensure GPIO 0 is grounded during upload, and that you have selected "AI Thinker ESP32-CAM" as the board.
- **Port not listed**: Install CP2102 drivers from https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers if using Windows.

---

## Notes for Instructors

- **Estimated build time**: 2-3 lab sessions for freshman students.
- **Key learning objectives**: UART communication, SPI protocol, GPIO configuration, task scheduling on dual-core microcontrollers, file system operations.
- **Common student pitfalls**: Reversing UART TX/RX connections, forgetting common ground, incorrect RFID UID entry, SD card not formatted as FAT32.
- **Extension ideas**: Add WiFi to upload photos to a server, implement a keypad for PIN-based arming, add a buzzer for audible alarm.

---

This tutorial is provided in Markdown format for easy editing. You can convert it to DOCX, PDF, or HTML using standard tools. All code references match the uploaded `main_board.cpp` and `cam_board.cpp` files.