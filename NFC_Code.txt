/*
 * UniTap POS Terminal - Heartbeat Architecture Implementation
 * Hardware: ESP32, 1.3" SH1106 OLED (I2C 21/47), MFRC522 (SPI), Buzzer (4), Boot Button (0)
 */

#include <Wire.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SH110X.h>
#include <SPI.h>
#include <MFRC522.h>

// ==========================================
// 1. GLOBAL CONFIGURATION (From Implementation Guide)
// ==========================================
const char* WIFI_SSID = "Zain";
const char* WIFI_PASS = "123zainu";
const char* API_BASE_URL = "https://unitap.refinedev.org/api/terminal";
const char* API_KEY = "your_esp32_api_key_here";
const char* TERMINAL_ID = "CANTEEN-01";

float currentPendingAmount = 0.0;
String currentScannedUID = ""; // Temporarily hold UID before processing

// ==========================================
// 2. STATE MACHINE SETUP
// ==========================================
enum TerminalState { 
  STATE_IDLE, 
  STATE_WAITING_FOR_TAP, 
  STATE_PROCESSING_TAP 
};

TerminalState currentState = STATE_IDLE;
unsigned long lastHeartbeatTime = 0;
const unsigned long HEARTBEAT_INTERVAL = 2000; // 2 seconds

// --- HARDWARE PINS ---
#define SDA_PIN 21
#define SCL_PIN 47
#define BUZZER_PIN 4
#define BOOT_BUTTON 0

// --- RFID PINS ---
#define RST_PIN         5          
#define SS_PIN          10         
MFRC522 mfrc522(SS_PIN, RST_PIN);  

#define i2c_Address 0x3c
Adafruit_SH1106G display = Adafruit_SH1106G(128, 64, &Wire, -1);

// Function Prototypes
void pollHeartbeat();
void processPayment(String uid);
String generateUUID();
void drawDynamicScreen(bool blinkText);
void showProcessingAnimation();
void showSuccessScreen(String name, String msg);
void showErrorScreen(String errorType, String msg);
void centerText(String text, int size, int y);
void playTone(int freq, int duration);

// ==========================================
// ARDUINO SETUP
// ==========================================
void setup() {
  Serial.begin(115200);
  pinMode(BOOT_BUTTON, INPUT_PULLUP);

  // Setup Buzzer
  ledcSetup(0, 2500, 8);
  ledcAttachPin(BUZZER_PIN, 0);

  // Init SPI and RFID
  SPI.begin(12, 13, 11, 10); // SCK, MISO, MOSI, SS
  mfrc522.PCD_Init();

  byte v = mfrc522.PCD_ReadRegister(mfrc522.VersionReg);
  Serial.print(F("RC522 Software Version: 0x"));
  Serial.println(v, HEX);
  if (v == 0x00 || v == 0xFF) {
    Serial.println(F("WARNING: Communication failure! Check RFID wiring."));
  }

  // Initialize Screen
  Wire.begin(SDA_PIN, SCL_PIN);
  display.begin(i2c_Address, true);

  showBootLogo();
  connectWiFi();
  
  // Enter initial state
  currentState = STATE_IDLE;
}

// ==========================================
// ARDUINO MAIN LOOP (STATE MACHINE)
// ==========================================
void loop() {
  // UI Animation timer
  static unsigned long lastUIUpdate = 0;
  static bool blinkToggle = true;

  if (millis() - lastUIUpdate > 800) {
    drawDynamicScreen(blinkToggle);
    blinkToggle = !blinkToggle;
    lastUIUpdate = millis();
  }

  // --- STATE MACHINE LOGIC ---
  switch (currentState) {
    
    // STATE 1: IDLE (Polling for orders)
    case STATE_IDLE:
      if (millis() - lastHeartbeatTime >= HEARTBEAT_INTERVAL) {
        lastHeartbeatTime = millis();
        pollHeartbeat(); 
      }
      break;

    // STATE 2: WAITING FOR TAP (Order primed)
    case STATE_WAITING_FOR_TAP:
      // 2a. Check Hardware NFC
      if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
        String uid = "";
        for (byte i = 0; i < mfrc522.uid.size; i++) {
          uid += String(mfrc522.uid.uidByte[i] < 0x10 ? "0" : "");
          uid += String(mfrc522.uid.uidByte[i], HEX);
        }
        uid.toUpperCase(); // Clean, uppercase Hex string with NO spaces/colons as per docs
        
        Serial.println("Card Detected: " + uid);
        playTone(3000, 100); // Success beep

        currentScannedUID = uid;
        currentState = STATE_PROCESSING_TAP;
        
        // Halt RFID to prevent rapid rescans
        mfrc522.PICC_HaltA();
        mfrc522.PCD_StopCrypto1();
      }

      // 2b. Fallback Trigger (Boot Button) - Only works if an order is primed
      if (digitalRead(BOOT_BUTTON) == LOW) {
        Serial.println("Simulating Tap via Boot Button");
        playTone(3000, 100);
        currentScannedUID = "04A3B2C1D4E5F6"; // Clean simulated UID
        currentState = STATE_PROCESSING_TAP;
        delay(500); // debounce
      }
      break;

    // STATE 3: PROCESSING TAP (Sending to backend)
    case STATE_PROCESSING_TAP:
      processPayment(currentScannedUID);
      break;
  }
}

// ==========================================
// CORE LOGIC: HEARTBEAT POLLING (STATE 1)
// ==========================================
void pollHeartbeat() {
  if (WiFi.status() != WL_CONNECTED) return;

  WiFiClientSecure client;
  client.setInsecure(); // Needed for HTTPS connections
  HTTPClient http;
  
  String url = String(API_BASE_URL) + "/heartbeat";
  http.begin(client, url);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("X-Terminal-Key", API_KEY);

  // Request Body
  StaticJsonDocument<200> reqDoc;
  reqDoc["terminalId"] = TERMINAL_ID;
  reqDoc["isOnline"] = true;

  String jsonPayload;
  serializeJson(reqDoc, jsonPayload);
  
  int httpCode = http.POST(jsonPayload);

  if (httpCode > 0) {
    String response = http.getString();
    DynamicJsonDocument doc(512);
    DeserializationError error = deserializeJson(doc, response);

    if (!error && doc.containsKey("pendingAmount")) {
      float incomingAmount = doc["pendingAmount"];
      
      if (incomingAmount > 0.0) {
        currentPendingAmount = incomingAmount;
        Serial.print("Order received! Waiting for NFC Tap for Rs ");
        Serial.println(currentPendingAmount);
        
        playTone(2000, 150); // Friendly alert that order is primed
        currentState = STATE_WAITING_FOR_TAP; // Transition State
      }
    }
  }
  http.end();
}

// ==========================================
// CORE LOGIC: PAYMENT PROCESSING (STATE 3)
// ==========================================
void processPayment(String uid) {
  if (WiFi.status() != WL_CONNECTED) {
    showErrorScreen("NETWORK ERR", "No WiFi");
    currentState = STATE_IDLE; // Fail gracefully
    return;
  }

  showProcessingAnimation();

  WiFiClientSecure client;
  client.setInsecure(); // Needed for HTTPS connections
  HTTPClient http;
  
  String url = String(API_BASE_URL) + "/tap";
  http.begin(client, url);
  http.addHeader("Content-Type", "application/json");
  http.addHeader("X-Terminal-Key", API_KEY);

  // JSON Request Body fulfilling Idempotency Requirements
  StaticJsonDocument<256> reqDoc;
  reqDoc["cardUid"] = uid;
  reqDoc["terminalId"] = TERMINAL_ID;
  reqDoc["amount"] = currentPendingAmount;
  reqDoc["transactionUuid"] = generateUUID(); 
  reqDoc["timestamp"] = 0; // Or standard timestamp if NTP is implemented

  String json;
  serializeJson(reqDoc, json);
  int httpCode = http.POST(json);

  if (httpCode > 0) {
    String response = http.getString();
    DynamicJsonDocument doc(1024);
    deserializeJson(doc, response);

    bool success = doc["success"];
    String msg = doc["message"] | "Processed";
    String name = doc["userName"] | "User";

    if (success) {
      Serial.println("Paid! New Balance: " + doc["newBalance"].as<String>());
      showSuccessScreen(name, msg);
    } else {
      showErrorScreen("DECLINED", msg);
    }
  } else {
    showErrorScreen("API ERROR", "Code " + String(httpCode));
  }

  http.end();

  // Reset System logic per Implementation Guide
  currentPendingAmount = 0.0;
  delay(3000); // 3 second delay so user can remove card
  display.clearDisplay(); 
  
  // Return to idle polling
  currentState = STATE_IDLE;
}

// Helper function to generate a unique UUID for idempotency
String generateUUID() {
  char uuid[37];
  sprintf(uuid, "%08lX-%04lX-4%03lX-%04lX-%08lX%04lX",
          esp_random(), esp_random() % 0xFFFF, esp_random() % 0xFFF, 
          (esp_random() % 0x3FFF) | 0x8000, esp_random(), esp_random() % 0xFFFF);
  return String(uuid);
}

// ==========================================
// UI DRAWING FUNCTIONS
// ==========================================
void drawDynamicScreen(bool blinkText) {
  // 1. Top Bar
  display.fillRect(0, 0, 128, 14, SH110X_WHITE);
  display.setTextSize(1); 
  display.setTextColor(SH110X_BLACK);
  display.setCursor(2, 3);
  display.print("UniTap PoS");

  // 2. WiFi Icon
  display.setTextColor(SH110X_WHITE); 
  display.fillCircle(118, 8, 1, SH110X_BLACK);
  display.drawCircleHelper(118, 8, 4, 3, SH110X_BLACK);
  display.drawCircleHelper(118, 8, 7, 3, SH110X_BLACK);

  // 3. Main Content
  display.fillRect(0, 14, 128, 50, SH110X_BLACK); 

  if (currentState == STATE_IDLE) {
    // Waiting for Server
    centerText("Waiting for", 1, 25);
    if (blinkText) centerText("Order...", 1, 40);
  } 
  else if (currentState == STATE_WAITING_FOR_TAP) {
    // Order Received
    String priceText = "Rs. " + String(currentPendingAmount, 2);
    centerText(priceText, 2, 25); 
    
    if (blinkText) {
      centerText("Please Tap Card", 1, 50);
    }
  }
  display.display();
}

// ... [Remaining UI Helper functions: showBootLogo, connectWiFi, etc] ...
void centerText(String text, int size, int y) {
  display.setTextSize(size);
  int textWidth = text.length() * (size * 6);
  int x = (128 - textWidth) / 2;
  display.setCursor(x, y);
  display.println(text);
}

void playTone(int freq, int duration) {
  ledcWriteTone(0, freq);
  delay(duration);
  ledcWriteTone(0, 0);
}

void showBootLogo() {
  display.clearDisplay();
  display.drawRect(0, 0, 128, 64, SH110X_WHITE);
  display.fillRect(0, 0, 128, 16, SH110X_WHITE);

  display.setTextColor(SH110X_BLACK);
  centerText("UniTap", 1, 4);

  display.setTextColor(SH110X_WHITE);
  centerText("PoS Terminal", 1, 25);
  centerText("Ready", 1, 40);
  display.display();
  delay(1500);
}

void connectWiFi() {
  display.clearDisplay();
  display.setTextColor(SH110X_WHITE);
  centerText("Connecting", 1, 20);
  centerText("to WiFi...", 1, 35);
  display.display();

  WiFi.begin(WIFI_SSID, WIFI_PASS);
  int dotCount = 0;
  while (WiFi.status() != WL_CONNECTED) {
    display.fillRect(30, 50, 68, 10, SH110X_BLACK); 
    String dots = "";
    for (int i = 0; i <= dotCount; i++) dots += ".";
    centerText(dots, 1, 50);
    display.display();
    dotCount = (dotCount + 1) % 4;
    delay(400);
  }
}

void showProcessingAnimation() {
  for (int i = 0; i <= 100; i += 20) {
    display.clearDisplay();
    centerText("Authorizing", 1, 20);

    // Draw Progress Bar
    display.drawRect(14, 40, 100, 10, SH110X_WHITE);
    display.fillRect(16, 42, (i * 96) / 100, 6, SH110X_WHITE);

    display.display();
    delay(100); 
  }
}

void showSuccessScreen(String name, String msg) {
  display.clearDisplay();

  // Draw Checkmark
  display.drawLine(50, 25, 60, 35, SH110X_WHITE);
  display.drawLine(50, 26, 60, 36, SH110X_WHITE);
  display.drawLine(60, 35, 80, 10, SH110X_WHITE);
  display.drawLine(60, 36, 80, 11, SH110X_WHITE);

  centerText("APPROVED", 1, 40);
  centerText(name, 1, 56);
  display.display();

  playTone(2000, 100);
  delay(50);
  playTone(3000, 200);
}

void showErrorScreen(String errorType, String msg) {
  display.invertDisplay(true);
  display.clearDisplay();

  // Draw an X
  display.drawLine(56, 22, 62, 28, SH110X_WHITE); 
  display.drawLine(56, 23, 62, 29, SH110X_WHITE); 
  display.drawLine(62, 28, 72, 14, SH110X_WHITE); 
  display.drawLine(62, 29, 72, 15, SH110X_WHITE); 

  centerText(errorType, 2, 42);
  centerText(msg, 1, 56);
  display.display();

  playTone(800, 800);
  display.invertDisplay(false);
}