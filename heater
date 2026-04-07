#include <Arduino.h>
#include <WiFi.h>

#include <SinricPro.h>
#include <SinricProSwitch.h>

#include <NimBLEDevice.h>
#include <string>

// =========================
// USER SETTINGS
// =========================
#define WIFI_SSID     "YOUR_WIFI_NAME"
#define WIFI_PASS     "YOUR_WIFI_PASSWORD"

#define APP_KEY       "YOUR_APP_KEY"
#define APP_SECRET    "YOUR_APP_SECRET"
#define DEVICE_ID     "YOUR_SWITCH_DEVICE_ID"

// Heater BLE MAC
static const char *HEATER_ADDR = "YOUR_Heater_MAC";

// Timing
static const uint32_t CMD_DELAY_MS   = 650;
static const uint32_t STATUS_WAIT_MS = 1200;
static const uint32_t STATUS_POLL_MS = 20000;
static const uint32_t BLE_RETRY_MS   = 5000;

// =========================
// UUIDS
// =========================
static NimBLEUUID serviceUUID("0000fff0-0000-1000-8000-00805f9b34fb");
static NimBLEUUID notifyUUID ("0000fff1-0000-1000-8000-00805f9b34fb");
static NimBLEUUID writeUUID  ("0000fff2-0000-1000-8000-00805f9b34fb");

// =========================
// GLOBALS
// =========================
NimBLEClient* pClient = nullptr;
NimBLERemoteService* pService = nullptr;
NimBLERemoteCharacteristic* pNotifyChar = nullptr;
NimBLERemoteCharacteristic* pWriteChar = nullptr;

bool bleConnected = false;
bool heaterPowerState = false;
bool heaterHeatingState = false;
uint8_t heaterMode = 0;
int heaterSetTempF = 70;
bool gotFreshStatus = false;

unsigned long lastBleAttempt = 0;
unsigned long lastStatusPoll = 0;

// =========================
// HELPERS
// =========================
uint8_t checksum8(const uint8_t* data, size_t len) {
  uint16_t sum = 0;
  for (size_t i = 0; i < len; i++) sum += data[i];
  return (uint8_t)(sum & 0xFF);
}

void printHex(const uint8_t* data, size_t len) {
  for (size_t i = 0; i < len; i++) {
    if (data[i] < 0x10) Serial.print('0');
    Serial.print(data[i], HEX);
    if (i + 1 < len) Serial.print(' ');
  }
  Serial.println();
}

// =========================
// PROTOCOL
// =========================
void buildBAAB(uint8_t cmd, uint8_t d1, uint8_t d2, uint8_t d3, uint8_t out[8]) {
  out[0] = 0xBA;
  out[1] = 0xAB;
  out[2] = 0x04;
  out[3] = cmd;
  out[4] = d1;
  out[5] = d2;
  out[6] = d3;
  out[7] = checksum8(out, 7);
}

bool writeHeaterCommand(const uint8_t* payload, size_t len) {
  if (!bleConnected || !pWriteChar) {
    Serial.println("[BLE] not connected");
    return false;
  }

  bool ok = pWriteChar->writeValue(payload, len, true);
  if (ok) {
    Serial.print("[BLE] TX: ");
    printHex(payload, len);
  } else {
    Serial.println("[BLE] write failed");
  }
  return ok;
}

bool sendStatusRequest() {
  uint8_t cmd[8];
  buildBAAB(0xCC, 0x00, 0x00, 0x00, cmd);
  gotFreshStatus = false;
  return writeHeaterCommand(cmd, sizeof(cmd));
}

bool sendHeatOn() {
  uint8_t cmd[8];
  buildBAAB(0xBB, 0xA1, 0x00, 0x00, cmd);
  return writeHeaterCommand(cmd, sizeof(cmd));
}

bool sendVent() {
  uint8_t cmd[8];
  buildBAAB(0xBB, 0xA4, 0x00, 0x00, cmd);
  return writeHeaterCommand(cmd, sizeof(cmd));
}

bool heaterOffSequence() {
  // Best-known off sequence for your heater family
  if (!sendVent()) return false;
  delay(CMD_DELAY_MS);

  if (!sendHeatOn()) return false;
  delay(CMD_DELAY_MS);

  if (!sendVent()) return false;
  return true;
}

// =========================
// STATUS DECODE
// =========================
void decodeStatusFrame(const uint8_t* data, size_t len) {
  if (len < 7) return;
  if (data[0] != 0xAB || data[1] != 0xBA) return;
  if (data[3] != 0xCC) return;

  uint8_t state = data[4];
  heaterMode = data[5];
  heaterSetTempF = data[6];

  heaterHeatingState = (state == 0x01);
  heaterPowerState = (state != 0x00);
  gotFreshStatus = true;

  const char* stateText = "UNKNOWN";
  switch (state) {
    case 0x00: stateText = "OFF"; break;
    case 0x01: stateText = "HEATING"; break;
    case 0x02: stateText = "COOLING"; break;
    case 0x04: stateText = "VENT"; break;
  }

  Serial.printf("[HEATER] State=%s Mode=0x%02X SetTemp=%dF\n",
                stateText, heaterMode, heaterSetTempF);
}

// =========================
// BLE NOTIFY CALLBACK
// =========================
void notifyCB(
  NimBLERemoteCharacteristic* pRemoteCharacteristic,
  uint8_t* pData,
  size_t length,
  bool isNotify) {

  Serial.print("[BLE] RX: ");
  printHex(pData, length);
  decodeStatusFrame(pData, length);
}

// =========================
// BLE CONNECT
// =========================
bool connectToHeater() {
  Serial.printf("[BLE] Connecting to %s\n", HEATER_ADDR);

  if (pClient) {
    if (pClient->isConnected()) {
      pClient->disconnect();
    }
    NimBLEDevice::deleteClient(pClient);
    pClient = nullptr;
  }

  pClient = NimBLEDevice::createClient();
  if (!pClient) {
    Serial.println("[BLE] create client failed");
    bleConnected = false;
    return false;
  }

  NimBLEAddress heaterBleAddr(std::string(HEATER_ADDR), BLE_ADDR_PUBLIC);

  if (!pClient->connect(heaterBleAddr)) {
    Serial.println("[BLE] connect failed");
    bleConnected = false;
    return false;
  }

  pService = pClient->getService(serviceUUID);
  if (!pService) {
    Serial.println("[BLE] FFF0 service not found");
    pClient->disconnect();
    bleConnected = false;
    return false;
  }

  pNotifyChar = pService->getCharacteristic(notifyUUID);
  pWriteChar  = pService->getCharacteristic(writeUUID);

  if (!pNotifyChar || !pWriteChar) {
    Serial.println("[BLE] FFF1/FFF2 characteristic missing");
    pClient->disconnect();
    bleConnected = false;
    return false;
  }

  if (pNotifyChar->canNotify()) {
    if (!pNotifyChar->subscribe(true, notifyCB)) {
      Serial.println("[BLE] subscribe failed");
    }
  }

  bleConnected = true;
  Serial.println("[BLE] connected");

  delay(300);
  sendStatusRequest();
  delay(STATUS_WAIT_MS);
  return true;
}

bool ensureBleConnected() {
  if (pClient && pClient->isConnected() && bleConnected) return true;
  return connectToHeater();
}

// =========================
// HEATER CONTROL
// =========================
bool heaterPowerOn() {
  if (!ensureBleConnected()) return false;

  bool ok = sendHeatOn();
  delay(CMD_DELAY_MS);

  sendStatusRequest();
  delay(STATUS_WAIT_MS);

  return ok;
}

bool heaterPowerOff() {
  if (!ensureBleConnected()) return false;

  bool ok = heaterOffSequence();
  delay(CMD_DELAY_MS);

  sendStatusRequest();
  delay(STATUS_WAIT_MS);

  return ok;
}

// =========================
// SINRIC CALLBACK
// =========================
bool onPowerState(const String &deviceId, bool &state) {
  Serial.printf("[Sinric] Power request: %s\n", state ? "ON" : "OFF");

  bool ok = state ? heaterPowerOn() : heaterPowerOff();
  if (!ok) {
    Serial.println("[Sinric] power command failed");
    return false;
  }

  return true;
}

// =========================
// WIFI / SINRIC
// =========================
void setupWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);

  Serial.print("[WiFi] connecting");
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(500);
  }
  Serial.println();
  Serial.print("[WiFi] IP: ");
  Serial.println(WiFi.localIP());
}

void setupSinric() {
  SinricProSwitch &heater = SinricPro[DEVICE_ID];
  heater.onPowerState(onPowerState);

  SinricPro.begin(APP_KEY, APP_SECRET);
  Serial.println("[Sinric] started");
}

// =========================
// ARDUINO
// =========================
void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println();
  Serial.println("ESP32 Heater Alexa Switch Bridge");

  setupWiFi();

  NimBLEDevice::init("");
  NimBLEDevice::setPower(ESP_PWR_LVL_P9);

  connectToHeater();
  setupSinric();

  lastStatusPoll = millis();
}

void loop() {
  SinricPro.handle();

  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("[WiFi] disconnected, reconnecting...");
    WiFi.disconnect();
    WiFi.begin(WIFI_SSID, WIFI_PASS);
    delay(1000);
  }

  if ((!bleConnected || !pClient || !pClient->isConnected()) &&
      millis() - lastBleAttempt > BLE_RETRY_MS) {
    lastBleAttempt = millis();
    connectToHeater();
  }

  if (bleConnected && millis() - lastStatusPoll > STATUS_POLL_MS) {
    lastStatusPoll = millis();
    sendStatusRequest();
  }

  delay(10);
}
