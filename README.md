# 🌤️ IoT Weather Monitoring System

An IoT-based local weather station built using **ESP32**, **DHT22 sensor**, and **BMP280 sensor**.  
It monitors temperature, humidity, and barometric pressure, displaying everything on a stunning, locally hosted web dashboard.  
The system communicates entirely on your local Wi-Fi, without needing any external cloud services!

---

## 🧠 Description
This project continuously measures atmospheric parameters in real-time.  
Instead of pushing to a third-party server, the ESP32 hosts a **sleek dark-mode web server** right from its memory.  
When you access its IP address on your phone or computer, background JavaScript seamlessly pulls live sensor updates every 2.5 seconds without ever reloading the webpage.

---

## ⚙️ Components Used
- **ESP32 NodeMCU**
- **DHT22 Temperature & Humidity Sensor**
- **BMP280 Barometric Pressure Sensor**

---

## 🔌 Wiring Connections

| Component | ESP32 Pin | Description |
|------------|--------------|-------------|
| **DHT22** | 3V3 | Power (3.3V) |
|           | GPIO 26 | Data signal |
|           | GND | Ground |
| **BMP280**| 3V3 | Power (3.3V) |
|           | GND | Ground |
|           | GPIO 22 | Serial Clock (SCL) |
|           | GPIO 21 | Serial Data (SDA) |

---

## 🚀 Features
- **Local Web Server**: Hosts a beautiful dark-mode UI straight from the ESP32 flash memory.
- **Real-Time Data**: Uses the `fetch` API for live dashboard updates every 2.5s. 
- **Non-blocking Architecture**: Built around `millis()` instead of `delay()` to keep the web server snappy at all times.
- **Dual Sensing**: High-precision temperature, humidity, and atmospheric pressure readouts.
- **No Cloud Required**: Completely private and functional on your local home network.

---

## 🧾 Usage Instructions
1. Open the code below in **Arduino IDE**  
2. Install required libraries via Library Manager:
   - `DHT sensor library` (by Adafruit)
   - `Adafruit BMP280 Library` (by Adafruit)
3. Update the WiFi credentials (`ssid`, `password`) at the top of the code.  
4. Connect components according to the wiring table.  
5. Upload the code to your ESP32 board.  
6. Open your **Serial Monitor** (115200 baud) to find your ESP32's assigned IP Address.  
7. Type that exact IP address into your browser to view the live dashboard!

---

## 💻 Arduino Code

```cpp
#include <Wire.h>
#include <SPI.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BMP280.h>
#include <DHT.h>
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Vivo V70";
const char* password = "03012006";

WebServer server(80);

#define DHT22_PIN 26
#define DHT22_TYPE DHT22

DHT dht22(DHT22_PIN, DHT22_TYPE);
Adafruit_BMP280 bmp;

float temp22 = 0.0, hum22 = 0.0;
float tempBmp = 0.0, pressBmp = 0.0;
bool fail22 = true, failBmp = true;

unsigned long previousMillis = 0;
const long interval = 2500;

const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE HTML>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Weather Dashboard</title>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;800&display=swap" rel="stylesheet">
    <style>
        :root {
            --bg-color: #0f172a;
            --card-bg: rgba(30, 41, 59, 0.7);
            --card-border: rgba(255, 255, 255, 0.1);
            --text-main: #f8fafc;
            --text-muted: #94a3b8;
            --accent: #38bdf8;
            --error: #ef4444;
        }

        body {
            margin: 0;
            padding: 2rem;
            background-color: var(--bg-color);
            background-image: 
                radial-gradient(at 0% 0%, rgba(56, 189, 248, 0.15) 0px, transparent 50%),
                radial-gradient(at 100% 100%, rgba(139, 92, 246, 0.15) 0px, transparent 50%);
            background-attachment: fixed;
            color: var(--text-main);
            font-family: 'Inter', sans-serif;
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            align-items: center;
        }

        .container {
            max-width: 800px;
            width: 100%;
        }

        header {
            text-align: center;
            margin-bottom: 3rem;
        }

        h1 {
            font-size: 2.5rem;
            font-weight: 800;
            margin: 0;
            background: linear-gradient(to right, #38bdf8, #818cf8);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
        }

        p.subtitle {
            color: var(--text-muted);
            margin-top: 0.5rem;
        }

        .dashboard {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 1.5rem;
            justify-content: center;
        }

        .card {
            background: var(--card-bg);
            border: 1px solid var(--card-border);
            border-radius: 16px;
            padding: 1.5rem;
            backdrop-filter: blur(10px);
            -webkit-backdrop-filter: blur(10px);
            transition: transform 0.3s ease, box-shadow 0.3s ease;
        }

        .card:hover {
            transform: translateY(-5px);
            box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.3);
        }

        .card h2 {
            font-size: 1.25rem;
            margin: 0 0 1rem 0;
            color: var(--text-muted);
            text-transform: uppercase;
            letter-spacing: 1px;
            font-weight: 600;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .status-dot {
            width: 10px;
            height: 10px;
            border-radius: 50%;
            background-color: #22c55e;
            box-shadow: 0 0 10px #22c55e;
            transition: background-color 0.3s ease;
        }
        
        .status-dot.error {
            background-color: var(--error);
            box-shadow: 0 0 10px var(--error);
        }

        .data-row {
            display: flex;
            align-items: baseline;
            margin-bottom: 1rem;
        }

        .data-row:last-child {
            margin-bottom: 0;
        }

        .value {
            font-size: 2.5rem;
            font-weight: 800;
            margin-right: 0.5rem;
            color: var(--text-main);
            transition: color 0.3s ease;
        }

        .unit {
            font-size: 1.25rem;
            color: var(--text-muted);
        }

        .label {
            width: 80px;
            color: var(--accent);
            font-weight: 600;
        }

        .connecting {
            position: fixed;
            top: 10px;
            right: 20px;
            font-size: 0.85rem;
            color: var(--text-muted);
            opacity: 0.5;
            transition: opacity 0.3s;
        }
        .connecting.active { opacity: 1; color: var(--accent); }

        @media (max-width: 600px) {
            h1 { font-size: 2rem; }
            .value { font-size: 2rem; }
            body { padding: 1rem; }
            .label { width: 60px; }
        }
    </style>
</head>
<body>
    <div class="connecting" id="sync">Syncing...</div>
    <div class="container">
        <header>
            <h1>Weather Monitoring System</h1>
            <p class="subtitle">Live Telemetry from ESP32</p>
        </header>

        <div class="dashboard">
            <div class="card" id="card-dht22">
                <h2>DHT22 <div class="status-dot" id="dot-dht22"></div></h2>
                <div class="data-row">
                    <span class="label">Temp</span>
                    <span class="value" id="t22">--</span>
                    <span class="unit">°C</span>
                </div>
                <div class="data-row">
                    <span class="label">Hum</span>
                    <span class="value" id="h22">--</span>
                    <span class="unit">%</span>
                </div>
            </div>

            <div class="card" id="card-bmp">
                <h2>BMP280 <div class="status-dot" id="dot-bmp"></div></h2>
                <div class="data-row">
                    <span class="label">Temp</span>
                    <span class="value" id="tbmp">--</span>
                    <span class="unit">°C</span>
                </div>
                <div class="data-row">
                    <span class="label">Press</span>
                    <span class="value" id="pbmp">--</span>
                    <span class="unit">hPa</span>
                </div>
            </div>
        </div>
    </div>

    <script>
        function updateData() {
            const sync = document.getElementById('sync');
            sync.classList.add('active');
            
            fetch('/data')
                .then(response => response.json())
                .then(data => {
                    document.getElementById('t22').innerText = data.dht22.fail ? '--' : data.dht22.t.toFixed(1);
                    document.getElementById('h22').innerText = data.dht22.fail ? '--' : data.dht22.h.toFixed(1);
                    document.getElementById('dot-dht22').className = data.dht22.fail ? 'status-dot error' : 'status-dot';
                    
                    document.getElementById('tbmp').innerText = data.bmp.fail ? '--' : data.bmp.t.toFixed(1);
                    document.getElementById('pbmp').innerText = data.bmp.fail ? '--' : (data.bmp.p / 100.0).toFixed(1);
                    document.getElementById('dot-bmp').className = data.bmp.fail ? 'status-dot error' : 'status-dot';

                    setTimeout(() => sync.classList.remove('active'), 500);
                })
                .catch(err => {
                    console.error("Fetch error:", err);
                    document.querySelectorAll('.status-dot').forEach(el => el.className = 'status-dot error');
                    setTimeout(() => sync.classList.remove('active'), 500);
                });
        }

        updateData();
        setInterval(updateData, 2500);
    </script>
</body>
</html>
)rawliteral";

void handleRoot() {
  server.send(200, "text/html", index_html);
}

void handleData() {
  String json = "{";
  
  json += "\"dht22\":{";
  json += "\"fail\":" + String(fail22 ? "true" : "false") + ",";
  json += "\"t\":" + String(temp22) + ",";
  json += "\"h\":" + String(hum22);
  json += "},";
  
  json += "\"bmp\":{";
  json += "\"fail\":" + String(failBmp ? "true" : "false") + ",";
  json += "\"t\":" + String(tempBmp) + ",";
  json += "\"p\":" + String(pressBmp);
  json += "}";
  json += "}";

  server.send(200, "application/json", json);
}

void setup() {
  Serial.begin(115200);
  delay(1000); 

  WiFi.begin(ssid, password);

  // Initialize hardware I2C before sensors start
  Wire.begin(21, 22);

  dht22.begin();

  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    Serial.print(".");
    attempts++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.print("\nConnected! IP: ");
    Serial.println(WiFi.localIP());
  }

  if (!bmp.begin(0x76) && !bmp.begin(0x77)) {
    Serial.println("BMP280 init failed!");
    failBmp = true;
  } else {
    failBmp = false;
  }

  bmp.setSampling(Adafruit_BMP280::MODE_NORMAL,     
                  Adafruit_BMP280::SAMPLING_X2,     
                  Adafruit_BMP280::SAMPLING_X16,    
                  Adafruit_BMP280::FILTER_X16,      
                  Adafruit_BMP280::STANDBY_MS_500); 

  server.on("/", handleRoot);
  server.on("/data", handleData);
  server.begin();
}

void loop() {
  server.handleClient();

  unsigned long currentMillis = millis();
  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis;

    temp22 = dht22.readTemperature();
    hum22  = dht22.readHumidity();
    fail22 = isnan(temp22) || isnan(hum22);

    if (!failBmp) {
      tempBmp  = bmp.readTemperature();
      pressBmp = bmp.readPressure();
      if (isnan(tempBmp) || isnan(pressBmp)) {
        failBmp = true;
      }
    }
  }
}
```
