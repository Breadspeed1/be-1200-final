/*
 * ANTI-THEFT SYSTEM — Main ESP32
 * Central coordinator: reads sensors, buffers photo from CAM
 * Board: ESP32 Dev Module
 *
 * Core 1 (main loop): RFID, sensors, LED control — always responsive.
 * Core 0 (alarm task): Photo capture runs in background.
 *
 * UART0 (Serial)       = USB debug
 * UART2 (Serial2)      = ESP32-CAM (TX=17, RX=16)
 */

#include <MFRC522.h>
#include <SPI.h>

// #define VIBRATION_PIN 13
#define REED_PIN 14
#define CAM_TX 17
#define CAM_RX 16

#define RC_SDA (26) // Changed from 35 (Input-only) to 26
#define RC_SCK (32)
#define RC_MOS (33)
#define RC_IMI (25)
#define RC_RST (27) // Changed from 34 (Input-only) to 27

// Status LEDs
#define LED_GREEN 19
#define LED_RED 18

MFRC522 mfrc522(RC_SDA, RC_RST);

// Add your authorized UID here
byte authorizedUID[4] = {0xF9, 0x49, 0xB0,
						 0x05}; // Replace with your actual UID

volatile bool armed = false;
bool lastVibration = LOW;
bool lastReed = LOW;
unsigned long lastAlarmTime = 0;
#define DEBOUNCE_MS 5000

// Image buffer
#define IMG_BUF_SIZE 51200
uint8_t *imgBuffer = NULL;
size_t imgSize = 0;

// Alarm task control
volatile bool alarmInProgress = false;

// Forward declaration
void alarmTask(void *param);

// ── Receive Photo from CAM ──────────────────────────────────
bool receivePhotoFromCAM() {
	unsigned long timeout = millis() + 10000;
	bool gotHeader = false;
	size_t expectedSize = 0;

	while (millis() < timeout) {
		if (Serial2.available()) {
			if (!gotHeader) {
				String line = Serial2.readStringUntil('\n');
				line.trim();
				if (line.length() > 0) {
					Serial.println("  CAM: " + line);

					if (line.startsWith("IMG:")) {
						expectedSize = line.substring(4).toInt();
						if (expectedSize > 0 && expectedSize <= IMG_BUF_SIZE &&
							imgBuffer) {
							gotHeader = true;
							imgSize = 0;
							Serial.println("  Receiving " +
										   String(expectedSize) + " bytes...");
						} else {
							Serial.println("  Invalid image size: " +
										   String(expectedSize));
							return false;
						}
					}
				}
			} else {
				size_t avail = Serial2.available();
				size_t toRead = expectedSize - imgSize;
				if (avail < toRead)
					toRead = avail;
				if (toRead > 0) {
					Serial2.readBytes(imgBuffer + imgSize, toRead);
					imgSize += toRead;
				}

				if (imgSize >= expectedSize) {
					delay(100);
					while (Serial2.available()) {
						String end = Serial2.readStringUntil('\n');
						end.trim();
						if (end.length() > 0)
							Serial.println("  CAM: " + end);
					}
					Serial.println("  Image buffered (" + String(imgSize) +
								   " bytes)");
					return true;
				}
			}
		}
		vTaskDelay(pdMS_TO_TICKS(1)); // Yield briefly
	}

	if (gotHeader) {
		Serial.println("  Photo timeout (" + String(imgSize) + "/" +
					   String(expectedSize) + " bytes)");
	} else {
		Serial.println("  Photo timeout (no header)");
	}
	return false;
}

// ── Start Alarm (launches background task on Core 0) ────────
void startAlarm(String reason) {
	unsigned long now = millis();
	if (now - lastAlarmTime < DEBOUNCE_MS)
		return;
	lastAlarmTime = now;

	alarmInProgress = true;
	Serial.println("");
	Serial.println("*** ALARM TRIGGERED: " + reason + " ***");

	// Launch alarm task on Core 0 with 16KB stack
	char *reasonCopy = strdup(reason.c_str());
	xTaskCreatePinnedToCore(alarmTask,	 // function
							"AlarmTask", // name
							16384, // stack size (16KB — heavy String usage)
							reasonCopy, // parameter
							1,			// priority
							NULL,		// task handle
							0			// Core 0
	);
}

// ── Alarm Task (Core 0) — Runs in background ────────────────
void alarmTask(void *param) {
	String reason = (char *)param;
	free(param);

	Serial.println("  -> Requesting photo");
	Serial2.println("PHOTO");

	imgSize = 0;
	bool photoOK = receivePhotoFromCAM();

	if (photoOK && imgSize > 0) {
		Serial.println("  -> Photo received and buffered.");
	} else {
		Serial.println("  -> No photo available");
	}

	Serial.println("Alarm handled.\n");
	alarmInProgress = false;

	// Delete this task when done
	vTaskDelete(NULL);
}

void setup() {
	Serial.begin(115200);
	Serial.println("\n====================================");
	Serial.println("   Anti-Theft System");
	Serial.println("   Main ESP32");
	Serial.println("====================================\n");

	imgBuffer = (uint8_t *)malloc(IMG_BUF_SIZE);
	if (imgBuffer) {
		Serial.println("Image buffer allocated (50 KB)");
	} else {
		Serial.println("Image buffer allocation FAILED");
	}

	pinMode(REED_PIN, INPUT);

	pinMode(LED_GREEN, OUTPUT);
	pinMode(LED_RED, OUTPUT);
	digitalWrite(LED_GREEN, LOW);
	digitalWrite(LED_RED, HIGH);

	Serial2.setRxBufferSize(2048);
	Serial2.begin(115200, SERIAL_8N1, CAM_RX, CAM_TX);

	// Init SPI and MFRC522
	SPI.begin(RC_SCK, RC_IMI, RC_MOS, RC_SDA);
	mfrc522.PCD_Init();
	Serial.println("RFID scanner initialized.");

	Serial.println("UART2  ->  ESP32-CAM");

	// Wait for CAM_READY
	Serial.println("Waiting for peripherals...");
	bool camReady = false;
	unsigned long bootDeadline = millis() + 25000;
	while (millis() < bootDeadline && !camReady) {
		if (Serial2.available()) {
			String line = Serial2.readStringUntil('\n');
			line.trim();
			if (line.length() > 0) {
				Serial.println("  CAM: " + line);
				if (line == "CAM_READY")
					camReady = true;
			}
		}
		delay(10);
	}
	if (!camReady)
		Serial.println("  CAM did not report ready");

	Serial.println("System ready - DISARMED\n");
}

// ── Main Loop (Core 1) — Always responsive ──────────────────
void loop() {
	// RFID Scanner
	if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
		Serial.print("RFID Detected UID:");
		for (byte i = 0; i < mfrc522.uid.size; i++) {
			Serial.print(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " ");
			Serial.print(mfrc522.uid.uidByte[i], HEX);
		}
		Serial.println();

		// Light up both LEDs to indicate detection
		digitalWrite(LED_GREEN, HIGH);
		digitalWrite(LED_RED, HIGH);

		bool match = true;
		for (byte i = 0; i < 4; i++) {
			if (mfrc522.uid.uidByte[i] != authorizedUID[i]) {
				match = false;
				break;
			}
		}

		mfrc522.PICC_HaltA();
		mfrc522.PCD_StopCrypto1();

		if (match) {
			armed = !armed; // Toggle state
			if (armed) {
				digitalWrite(LED_GREEN, HIGH);
				digitalWrite(LED_RED, LOW);
				Serial.println("System armed via RFID");
			} else {
				digitalWrite(LED_GREEN, LOW);
				digitalWrite(LED_RED, HIGH);
				Serial.println("System disarmed via RFID");
			}
			delay(1000); // Simple debounce for card read
		} else {
			Serial.println("Unauthorized RFID card read");
			delay(1000);

			// Restore LED states if unauthorized
			if (armed) {
				digitalWrite(LED_GREEN, HIGH);
				digitalWrite(LED_RED, LOW);
			} else {
				digitalWrite(LED_GREEN, LOW);
				digitalWrite(LED_RED, HIGH);
			}
		}
	}

	if (!armed)
		return;

	// Sensors — only trigger if no alarm already in progress
	if (!alarmInProgress) {
		// bool vibration = digitalRead(VIBRATION_PIN);
		// if (vibration == HIGH && lastVibration == LOW)
		//  startAlarm("Vibration");
		// lastVibration = vibration;

		bool reed = digitalRead(REED_PIN);
		if (reed == HIGH && lastReed == LOW)
			startAlarm("Door Opened");
		lastReed = reed;
	}
}
