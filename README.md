# ESP32-CAM Donation-Based Cat Feeder (Arduino ESP32)

This project implements an **IoT cat feeder** that drops food either on a **schedule** or when a **bank donation** is received.  
It uses an **ESP32-CAM** as the main controller and web server, a **servo** to open/close the feeder, an **ultrasonic sensor (HC-SR04)** to measure food level in the box, and the **SEPAY API** to check the latest bank transactions. Donor names are collected and shown via a web API, and a QR image for donation can be uploaded to the device.

## Features

- **Live camera stream** from ESP32-CAM on a separate port (`/stream`).
- **Web dashboard** served from the ESP32-CAM:
  - Main page at `/` rendered from `index_html.h`.
  - Shows current food level percentage.
  - Shows recent **donor list**.
  - Allows uploading / deleting a QR image for donations (stored in SPIFFS as `uploaded.jpg`).
  - Allows setting **3 daily feeding time slots** for the servo.
- **Scheduled feeding**:
  - User can configure up to **3 feeding times** (`time1`, `time2`, `time3`) via `/setServoTimes`.
  - Times are stored in **NVS** (using `Preferences`), so they persist after reset.
  - At the configured times, the servo rotates to drop food and then returns to its idle position.
- **Donation-based feeding (SEPAY integration)**:
  - Periodically calls the **SEPAY API** to fetch the latest bank transactions.
  - Detects a **new transaction** by comparing `transaction_date` with the last stored date.
  - Parses the transaction content to extract the donor name.
  - Updates a **donor list** (max 5 recent donors).
  - When a new donation is detected, the servo rotates to drop extra food for the cat.
- **Food level monitoring**:
  - Uses an **HC-SR04 ultrasonic sensor** to measure the distance to the top of the food.
  - Computes the current food height and converts it to a **percentage** of the box height.
  - Exposes `/feedLevel` endpoint to return the food percentage as plain text.
- **File upload & SPIFFS storage**:
  - Allows uploading an image (`uploaded.jpg`) through `/upload` to show a QR code on the web UI.
  - Allows downloading the image via `/uploaded.jpg`.
  - Allows deleting the image via `/delete`.

## Hardware

- **Main controller**: ESP32-CAM module (AI Thinker)
- **Actuators & sensors**:
  - **Servo** on `SERVO_PIN` (e.g. SG90) – used to open/close feeder.
  - **HC-SR04 ultrasonic sensor**:
    - Trigger pin: `TRIG_PIN`
    - Echo pin: `ECHO_PIN`
- **Storage & connectivity**:
  - Wi-Fi connection (with optional static IP).
  - SPIFFS for storing uploaded QR image (`uploaded.jpg`).
- **Power**:
  - 5 V supply for ESP32-CAM and servo (servo should have sufficient current).
- Other: wires, food container, mechanical feeder mechanism.

Default pin definitions (from the sketch):

#define SERVO_PIN 13
Camera pins are defined via #define CAMERA_MODEL_AI_THINKER and #include "camera_pins.h".
Software & Tools
Platform: Arduino core for ESP32
Main libraries:
esp_camera.h – ESP32-CAM camera driver
WiFi.h – Wi-Fi connectivit
WebServer.h and esp_http_server.h – HTTP server and MJPEG stream
FS.h and SPIFFS.h – filesystem for uploads
ArduinoJson.h – parsing JSON from the SEPAY API
HTTPClient.h – HTTP client to call SEPAY REST API
Preferences.h – persistent storage (NVS) for feeding times
ESP32Servo.h – servo control
C++ <deque> – storing donor names
Time sync:
configTime() with NTP servers (pool.ntp.org, time.nist.gov) to get real time for feeding schedule.
IDE:
Arduino IDE, PlatformIO or VS Code with ESP32 support.
⚠️ Before publishing to GitHub, replace your real ssid, password, api_token and accountNumber with placeholders or move them into a separate secrets.h file.
Project Structure
esp32-cam-donation-cat-feeder/
├─ code/
│  ├─ cat_feeder_cam.ino   # ESP32-CAM: camera, Wi-Fi, web UI, servo, food level, SEPAY checks
│  ├─ index_html.h         # Web dashboard HTML (embedded as C string, optional/public or private)
├─ docs/
│  ├─ wiring_diagram.png   # System / wiring diagram (optional)
├─ README.md

Getting the Web UI Source (index_html.h)


Code Overview
1. cat_feeder_cam.ino (ESP32-CAM main controller)
Wi-Fi and time setup
-Configures a static IP using WiFi.config() (e.g. 192.168.1.200).
-Connects to Wi-Fi with SSID and password.
-Prints the local IP address to the serial monitor.
-Uses configTime() with a time zone offset and NTP servers to synchronize system time.
-Checks getLocalTime() to confirm time synchronization.

Servo and distance sensor
-Attaches a servo (camServo.attach(SERVO_PIN)) and sets initial angle to 90°.
-Initializes trigger/echo pins for HC-SR04:
-TRIG_PIN as output
-ECHO_PIN as input
-readDistanceCM():
-Sends a 10 µs pulse on TRIG.
-Uses pulseIn(ECHO_PIN, HIGH, 30000) to measure echo time.
-Converts duration to distance (cm).
-Periodically (every 2 seconds):
-Reads distance to compute food height:
-foodHeight = BOX_HEIGHT - distance
-Converts to percentage:
-lastFoodPercent = (foodHeight / BOX_HEIGHT) * 100.0
-Prints the result to the serial monitor.

Preferences & feeding times
-Uses Preferences prefs; with namespace "servo".
-loadServoTimesFromPrefs():
-Loads up to 3 time strings (time1, time2, time3) from NVS.
-saveServoTimesToPrefs():
-Writes the current servoTimes[] array back to NVS.
-handleSetServoTimes():
-HTTP handler for /setServoTimes.
-Reads time1, time2, time3 from query arguments.
-Updates servoTimes[], saves them to NVS, and returns a success message.

Camera and web server
Initializes the camera with a QVGA JPEG configuration.
Starts a streaming HTTP server on port 81:
/stream endpoint:
Uses esp_http_server with a multipart MJPEG stream.
Continuously grabs frames using esp_camera_fb_get() and sends them as chunks.
Starts a WebServer (port 80) for REST endpoints:
/:
Serves the main HTML page from index_html (defined in index_html.h).
Replaces %IP% placeholder with the actual IP for camera stream URL.
/setServoTimes:Calls handleSetServoTimes() to update feeding times.
/donors:Returns a JSON array of recent donor names.
/upload (POST + handleFileUpload()):Handles file upload (QR image), saves as /uploaded.jpg in SPIFFS.
/uploaded.jpg:Streams the stored QR image from SPIFFS (if it exists).
/delete:Deletes uploaded.jpg from SPIFFS.
/feedLevel:Returns lastFoodPercent as plain text.

SEPAY donation check (checkSEPAY())
Runs every checkInterval milliseconds (e.g. 6000 ms).
If Wi-Fi is connected:
Builds request URL:
https://my.sepay.vn/userapi/transactions/list?account_number=<accountNumber>&limit=5
Sends an HTTP GET with headers:
"Content-Type": "application/json"
"Authorization": "Bearer <api_token>"
On HTTP 200:
Parses JSON payload using ArduinoJson.
Reads the latest transaction_date field.
If this date differs from lastTransactionDate, then:
Update lastTransactionDate.
Clear donorList.
For each transaction:
Extract transaction_content.
Parse donor name between "IBFT " and " chuyen tien".
If not found, default to "Ẩn danh".
Push donor name into donorList (limit to 5 names).
Rotate the servo (e.g. to 0°, then back to 90°) to drop food and confirm a new donation.

Main loop
loop() continuously:
Calls server.handleClient() to process incoming HTTP requests.
Every 2 seconds:
Measures food level with readDistanceCM() and updates lastFoodPercent.
Every checkInterval:
Calls checkSEPAY() to detect new donations and update the donor list.
Periodically checks current time via getLocalTime():
Compares current HH:MM (nowHM) with each configured servoTimes[i].
If the time matches a configured feeding time and it has not been triggered for that slot yet:
Prints a message to serial: "Đến giờ ... → Servo quay!".
Rotates the servo to dispense food (e.g. 0° then back to 90°).

How It Works (High-Level)
1/Power on the ESP32-CAM feeder device.
2/The device:
Configures a static IP (optional) and connects to the Wi-Fi network.
Synchronizes its time via NTP.
Initializes the camera, servo, ultrasonic sensor, SPIFFS and NVS (Preferences).
Starts:
Port 81: streaming server at /stream.
Port 80: REST server with /, /setServoTimes, /donors, /upload, /uploaded.jpg, /delete, /feedLevel.
3/The web browser connects to the ESP32-CAM:
Loads the main HTML page (from index_html.h).
Shows the live camera stream and food level.
Allows:
Setting daily feeding times.
Viewing donor names.
Uploading/deleting a QR image for donations.
4/In the background:
Every few seconds:
The feeder measures the current food level using the ultrasonic sensor.
Every checkInterval ms:
It calls the SEPAY API to fetch the latest bank transactions.
When a new transaction appears:
Updates donor list.
Rotates the servo to drop additional food for the cat.
5/At configured feeding times:
The device checks current HH:MM against the saved servoTimes.
When a match is found for a given slot and it hasn’t been triggered yet that minute:
The servo rotates to dispense food and then returns to the idle angle.
This makes the ESP32-CAM Donation-Based Cat Feeder capable of both scheduled feeding and donation-triggered feeding, with live video monitoring, donor list, and food level feedback through a simple web interface.

For the full web dashboard source code (index_html.h), please contact me directly:
Email: letronghoang0604@gmail.com
Zalo: 0903240604
I will be happy to share the index_html.h file and instructions on how to integrate it into the project.

