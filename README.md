# ESP32-VEVOR-diesel-heater
VEVOR-diesel-heater on ESP32 for Alexia. 

1.  In Sinric Pro, make the device a plain Switch.

      Make Account/Login https://sinric.pro

    Use these Arduino IDE settings:

          Board: ESP32 Dev Module
          Flash Size: 4MB
          Partition Scheme: Huge APP
          Core Debug Level: None

What works:

     “Alexa, turn heater on”
     “Alexa, turn heater off”

 =========================
// USER SETTINGS
// =========================

    #define WIFI_SSID     "YOUR_WIFI_NAME"
    #define WIFI_PASS     "YOUR_WIFI_PASSWORD"

    #define APP_KEY       "YOUR_APP_KEY"
    #define APP_SECRET    "YOUR_APP_SECRET"
    #define DEVICE_ID     "YOUR_SWITCH_DEVICE_ID"

=========================
// Heater BLE MAC
// =========================
          
    static const char *HEATER_ADDR = "YOUR_Heater_MAC";

 =========================
// Bluetooth UUIDS
// =========================

    static NimBLEUUID serviceUUID("0000fff0-0000-1000-8000-00805f9b34fb");
    static NimBLEUUID notifyUUID ("0000fff1-0000-1000-8000-00805f9b34fb");
    static NimBLEUUID writeUUID  ("0000fff2-0000-1000-8000-00805f9b34fb");
