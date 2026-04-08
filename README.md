# 🌿 Smart Greenhouse Monitoring and Alert System

An IoT-based smart greenhouse system built using **ESP8266 NodeMCU**, **DHT11 sensor**, **Soil Moisture sensor**, and **Buzzer**.  
It monitors temperature, humidity, and soil moisture, providing alerts when thresholds are exceeded.  
The system also uploads real-time data to **ThingSpeak** for remote IoT monitoring.

---

## 🧠 Description
This project continuously measures environmental parameters crucial for plant growth.  
Whenever the soil is dry, temperature too high or low, or humidity out of range, the **buzzer alerts** the user.  
At the same time, data is uploaded to the **ThingSpeak Cloud**, enabling remote monitoring via IoT.

---

## ⚙️ Components Used
- **ESP8266 NodeMCU**
- **DHT11 Temperature & Humidity Sensor**
- **Soil Moisture Sensor**
- **Buzzer (Alert System)**

---

## 🔌 Wiring Connections

| Component | NodeMCU Pin | Description |
|------------|--------------|-------------|
| DHT11 Sensor | D2 | Data pin |
| Soil Moisture Sensor | A0 | Analog output |
| Buzzer | D5 | Positive terminal |
| Common Ground | G | All GND connections |

---

## 🚀 Features
- Real-time monitoring of **temperature**, **humidity**, and **soil moisture**
- **Automatic buzzer alerts** for dry soil or abnormal climate conditions
- **ThingSpeak cloud integration** for IoT data logging and visualization
- Easy-to-read **Serial Monitor output**

---

## 🧾 Usage Instructions
1. Open the code below in **Arduino IDE**  
2. Install required libraries:
   - `ESP8266WiFi.h`
   - `ThingSpeak.h`
   - `DHT.h`
3. Update the WiFi credentials (`ssid`, `password`) and ThingSpeak details (`channelID`, `writeAPIKey`)  
4. Connect components according to the wiring table  
5. Upload the code to your NodeMCU board  
6. Open Serial Monitor and observe live data  
7. Check data remotely on your **ThingSpeak dashboard**

---

## 💻 Arduino Code

```cpp
#include <ESP8266WiFi.h>
#include <ThingSpeak.h>
#include "DHT.h"

// ---------- PIN DEFINITIONS ----------
#define DHTPIN D2           // DHT11 Data Pin
#define DHTTYPE DHT11
#define SOILPIN A0          // Soil Moisture Analog Pin
#define BUZZER D5           // Buzzer Pin

// ---------- WiFi & ThingSpeak ----------
const char* ssid = ""; 
const char* password = ""; 

unsigned long channelID = 3111183;            // ThingSpeak channel ID
const char* writeAPIKey = "56M44PYUELKEI77T"; // ThingSpeak read API key

WiFiClient client;
DHT dht(DHTPIN, DHTTYPE);

unsigned long lastDHTRead = 0;
unsigned long lastThingSpeakUpdate = 0;

void setup() {
  Serial.begin(115200);
  dht.begin();
  pinMode(BUZZER, OUTPUT);

  Serial.println("🌱 Smart Greenhouse System Starting...");

  // Connect to WiFi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\n✅ WiFi Connected!");
  ThingSpeak.begin(client);
}

void loop() {
  bool buzzerState = LOW;  // Default buzzer OFF

  // ----- SOIL MOISTURE SENSOR -----
  int soilValue = analogRead(SOILPIN);
  Serial.print("🌱 Soil Moisture Value: ");
  Serial.println(soilValue);

  if (soilValue > 800) {
    Serial.println("⚠️ Soil is very dry!");
    buzzerState = HIGH;
  } else if (soilValue > 500) {
    Serial.println("🙂 Soil is moist.");
  } else {
    Serial.println("💧 Soil is wet.");
  }

  // ----- DHT11 SENSOR every 3 sec -----
  if (millis() - lastDHTRead > 3000) {
    lastDHTRead = millis();
    float h = dht.readHumidity();
    float t = dht.readTemperature();

    if (isnan(h) || isnan(t)) {
      Serial.println("❌ Failed to read from DHT sensor!");
    } else {
      Serial.print("🌡 Temperature: ");
      Serial.print(t);
      Serial.print("°C, Humidity: ");
      Serial.print(h);
      Serial.println("%");

      // ----- TEMPERATURE ALERT -----
      if (t > 32) {
        Serial.println("🔥 High Temperature Alert!");
        buzzerState = HIGH;
      } else if (t < 18) {
        Serial.println("❄️ Low Temperature Alert!");
        buzzerState = HIGH;
      }

      // ----- HUMIDITY ALERT -----
      if (h < 40) {
        Serial.println("💨 Low Humidity Alert!");
        buzzerState = HIGH;
      } else if (h > 100) {
        Serial.println("💦 High Humidity Alert!");
        buzzerState = HIGH;
      }
    }
    Serial.println("-----------------------------");
  }

  // ----- Set Buzzer State -----
  digitalWrite(BUZZER, buzzerState);

  // ----- ThingSpeak update every 20 sec -----
  if (millis() - lastThingSpeakUpdate > 20000) {
    lastThingSpeakUpdate = millis();
    if (WiFi.status() == WL_CONNECTED) {
      ThingSpeak.setField(1, dht.readTemperature());
      ThingSpeak.setField(2, dht.readHumidity());
      ThingSpeak.setField(3, soilValue);
      int response = ThingSpeak.writeFields(channelID, writeAPIKey);
      if (response == 200) {
        Serial.println("✅ ThingSpeak Updated");
      } else {
        Serial.print("❌ ThingSpeak update failed, code: ");
        Serial.println(response);
      }
    } else {
      Serial.println("❌ WiFi not connected, cannot update ThingSpeak");
    }
  }

  delay(1000); // Sensor updates every second
}
