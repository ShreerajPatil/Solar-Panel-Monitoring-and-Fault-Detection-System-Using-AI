**Code**
```
#include <Wire.h>
#include <SPI.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_ILI9341.h>
#include <DHT.h>
#include <WiFi.h>
#include <PubSubClient.h>

// ─── OLED Config ────────────────────────────────────────────
#define SDA_PIN        8
#define SCL_PIN        9
#define SCREEN_WIDTH   128
#define SCREEN_HEIGHT  64
#define OLED_RESET     -1
#define SCREEN_ADDRESS 0x3C

// ─── ILI9341 — ESP32-S3 FSPI ────────────────────────────────
#define TFT_CS         10
#define TFT_RST        21
#define TFT_DC         14
#define TFT_MOSI       11
#define TFT_CLK        12
#define TFT_MISO       13

// ─── Voltage Pins ───────────────────────────────────────────
#define VOLTAGE_PIN_1  1
#define VOLTAGE_PIN_2  2
#define VOLTAGE_PIN_3  3

// ─── DHT11 Pins ─────────────────────────────────────────────
#define DHT_PIN_1      15
#define DHT_PIN_2      16
#define DHT_PIN_3      17
#define DHTTYPE        DHT11

// ─── Voltage Divider ────────────────────────────────────────
#define ADC_MAX        4095.0
#define ADC_VREF       3.3
#define R1             10000.0
#define R2             68000.0

// ─── Thresholds ─────────────────────────────────────────────
#define VOLTAGE_ZERO      0.5
#define VOLTAGE_LOW       10.0
#define TEMP_HOT          50.0
#define NOMINAL_VOLTAGE   24.0

// ─── Timing ─────────────────────────────────────────────────
#define NORMAL_DISPLAY_MS  3000
#define FAULT_DISPLAY_MS   5000
#define ANIM_FRAME_MS      250

// ─── ILI9341 Colors ─────────────────────────────────────────
#define TFT_BG      0x0841
#define TFT_GOLD    0xFEA0
#define TFT_GREEN   0x07E0
#define TFT_YELLOW  0xFFE0
#define TFT_RED     0xF800
#define TFT_WHITE   0xFFFF
#define TFT_DGRAY   0x4208
#define TFT_LGRAY   0xBDF7
#define TFT_ORANGE  0xFD20
#define TFT_CYAN    0x07FF
#define TFT_DRED    0x2800
#define TFT_DGREEN  0x0300

// ─── WiFi + MQTT Config ─────────────────────────────────────
const char* WIFI_SSID    = "CirkitWifi";
const char* WIFI_PASS    = "";
const char* MQTT_BROKER  = "broker.hivemq.com";
const int   MQTT_PORT    = 1883;
// ⚠ Change "yourname" to something unique (e.g. your name + digits)
const char* MQTT_CLIENT_ID = "solar_esp32_Shreeraj";
#define TOPIC_BASE "solar/fault/"

WiFiClient   wifiClient;
PubSubClient mqttClient(wifiClient);

// ════════════════════════════════════════════════════════════
SPIClass fspi(FSPI);
Adafruit_SSD1306  display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
Adafruit_ILI9341  tft(&fspi, TFT_DC, TFT_CS, TFT_RST);

DHT dht1(DHT_PIN_1, DHTTYPE);
DHT dht2(DHT_PIN_2, DHTTYPE);
DHT dht3(DHT_PIN_3, DHTTYPE);

// ════════════════════════════════════════════════════════════
// WiFi + MQTT
// ════════════════════════════════════════════════════════════
void connectWiFi() {
  WiFi.mode(WIFI_STA);
  Serial.print("[WiFi] Connecting to CirkitWifi");
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  int tries = 0;
  while (WiFi.status() != WL_CONNECTED && tries < 30) {
    delay(500); Serial.print("."); tries++;
  }
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println(" Connected!");
    Serial.print("[WiFi] IP: "); Serial.println(WiFi.localIP());
  } else {
    Serial.println(" FAILED (continuing without WiFi)");
  }
}

void connectMQTT() {
  if (WiFi.status() != WL_CONNECTED) return;
  mqttClient.setServer(MQTT_BROKER, MQTT_PORT);
  int tries = 0;
  while (!mqttClient.connected() && tries < 5) {
    Serial.print("[MQTT] Connecting...");
    if (mqttClient.connect(MQTT_CLIENT_ID)) {
      Serial.println(" Connected!");
    } else {
      Serial.print(" Failed rc="); Serial.print(mqttClient.state());
      Serial.println(" retrying in 2s");
      delay(2000); tries++;
    }
  }
}

void publishAllPanels(float v1, float t1, float h1, String f1, float e1,
                      float v2, float t2, float h2, String f2, float e2,
                      float v3, float t3, float h3, String f3, float e3) {
  if (!mqttClient.connected()) connectMQTT();
  if (!mqttClient.connected()) return;

  float  vArr[3] = {v1, v2, v3};
  float  tArr[3] = {t1, t2, t3};
  float  hArr[3] = {h1, h2, h3};
  float  eArr[3] = {e1, e2, e3};
  String fArr[3] = {f1, f2, f3};

  for (int i = 0; i < 3; i++) {
    char topic[48];
    snprintf(topic, sizeof(topic), TOPIC_BASE "panel%d", i + 1);

    char payload[120];
    snprintf(payload, sizeof(payload),
      "{\"v\":%.2f,\"t\":%.1f,\"h\":%.1f,\"eff\":%d,\"fault\":\"%s\"}",
      vArr[i], tArr[i], hArr[i], (int)eArr[i], fArr[i].c_str());

    mqttClient.publish(topic, payload);
    Serial.print("[MQTT] "); Serial.print(topic);
    Serial.print(" -> "); Serial.println(payload);
  }
  mqttClient.loop();
}

// ════════════════════════════════════════════════════════════
// SENSOR READS
// ════════════════════════════════════════════════════════════
float readVoltage(int pin) {
  long sum = 0;
  for (int i = 0; i < 16; i++) { sum += analogRead(pin); delay(2); }
  float avg        = sum / 16.0;
  float adcVoltage = (avg / ADC_MAX) * ADC_VREF;
  return adcVoltage * ((R1 + R2) / R1);
}

float readTemp(DHT &dht) {
  float t = dht.readTemperature();
  if (isnan(t)) { delay(1000); t = dht.readTemperature(); }
  return isnan(t) ? 0.0 : t;
}

float readHum(DHT &dht) {
  float h = dht.readHumidity();
  if (isnan(h)) { delay(1000); h = dht.readHumidity(); }
  return isnan(h) ? 0.0 : h;
}

// ════════════════════════════════════════════════════════════
// FAULT LOGIC
// ════════════════════════════════════════════════════════════
String getFaultCode(float voltage, float temp) {
  if (voltage < VOLTAGE_ZERO) return "F2";
  if (voltage < VOLTAGE_LOW)  return "F1";
  if (temp   >= TEMP_HOT)     return "F3";
  return "OK";
}

String getFaultMsg(String code) {
  if (code == "F1") return "Dust / Sand";
  if (code == "F2") return "Panel Dead";
  if (code == "F3") return "Hotspot!";
  return "Normal";
}

uint16_t effColor(float eff) {
  if (eff >= 70.0) return TFT_GREEN;
  if (eff >= 40.0) return TFT_YELLOW;
  return TFT_RED;
}

float calcEfficiency(float voltage) {
  return constrain((voltage / NOMINAL_VOLTAGE) * 100.0, 0.0, 100.0);
}

// ════════════════════════════════════════════════════════════
// OLED — EFFICIENCY SCREEN
// ════════════════════════════════════════════════════════════
void showEfficiencyOLED(float v1, float v2, float v3,
                        String f1, String f2, String f3) {
  float e1 = calcEfficiency(v1);
  float e2 = calcEfficiency(v2);
  float e3 = calcEfficiency(v3);

  display.clearDisplay();

  display.fillRect(0, 0, 128, 11, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setTextSize(1);
  display.setCursor(18, 2);
  display.print("PANEL EFFICIENCY");

  int rowY[3]     = {16, 32, 48};   // slightly more spread out now
  float effs[3]   = {e1, e2, e3};
  String codes[3] = {f1, f2, f3};

  for (int i = 0; i < 3; i++) {
    float  eff  = effs[i];
    String code = codes[i];
    int    y    = rowY[i];

    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);

    display.setCursor(0, y);
    display.print("P"); display.print(i + 1);

    int barX = 14, barW = 76, barH = 7;
    display.drawRect(barX, y, barW, barH, SSD1306_WHITE);

    int fillW = (int)((eff / 100.0) * (barW - 2));
    if (fillW > 0)
      display.fillRect(barX + 1, y + 1, fillW, barH - 2, SSD1306_WHITE);

    display.setCursor(92, y);
    display.print((int)eff); display.print("%");

    if (code == "OK") {
      display.fillRect(121, y + 1, 6, 6, SSD1306_WHITE);
    } else {
      display.drawRect(121, y + 1, 6, 6, SSD1306_WHITE);
      display.drawLine(121, y + 1, 126, y + 6, SSD1306_WHITE);
      display.drawLine(126, y + 1, 121, y + 6, SSD1306_WHITE);
    }
  }

  display.display();
}

// ════════════════════════════════════════════════════════════
// LCD — NORMAL PANEL READINGS
// ════════════════════════════════════════════════════════════
void showNormalLCD(int panelNum, float voltage,
                   float temp, float hum, float eff) {
  tft.fillScreen(TFT_BG);

  tft.fillRect(0, 0, 240, 30, TFT_GOLD);
  tft.setTextColor(0x0000); tft.setTextSize(1);
  tft.setCursor(12, 5);  tft.print("SOLAR FAULT DETECTION SYSTEM");
  tft.setCursor(12, 18); tft.print("ESP32-S3  |  3-Panel Monitor");

  tft.fillRoundRect(60, 36, 120, 28, 6, TFT_DGREEN);
  tft.drawRoundRect(60, 36, 120, 28, 6, TFT_GREEN);
  tft.setTextColor(TFT_GREEN); tft.setTextSize(2);
  tft.setCursor(78, 43);
  tft.print("PANEL "); tft.print(panelNum);

  tft.setTextColor(TFT_GREEN); tft.setTextSize(1);
  tft.setCursor(76, 74); tft.print("[ NO FAULT ]");

  tft.drawFastHLine(10, 88, 220, TFT_DGRAY);

  tft.fillRoundRect(10, 96, 220, 52, 6, 0x1082);
  tft.drawRoundRect(10, 96, 220, 52, 6, TFT_CYAN);
  tft.setTextColor(TFT_LGRAY); tft.setTextSize(1);
  tft.setCursor(20, 104); tft.print("VOLTAGE");
  tft.setTextColor(TFT_CYAN); tft.setTextSize(3);
  tft.setCursor(20, 116);
  tft.print(voltage, 1); tft.print(" V");

  tft.fillRoundRect(10, 156, 105, 52, 6, 0x1082);
  tft.drawRoundRect(10, 156, 105, 52, 6, TFT_ORANGE);
  tft.setTextColor(TFT_LGRAY); tft.setTextSize(1);
  tft.setCursor(20, 164); tft.print("TEMPERATURE");
  tft.setTextColor(TFT_ORANGE); tft.setTextSize(2);
  tft.setCursor(20, 178);
  tft.print(temp, 1); tft.print("C");

  tft.fillRoundRect(125, 156, 105, 52, 6, 0x1082);
  tft.drawRoundRect(125, 156, 105, 52, 6, TFT_CYAN);
  tft.setTextColor(TFT_LGRAY); tft.setTextSize(1);
  tft.setCursor(135, 164); tft.print("HUMIDITY");
  tft.setTextColor(TFT_CYAN); tft.setTextSize(2);
  tft.setCursor(135, 178);
  tft.print(hum, 1); tft.print("%");

  tft.drawFastHLine(10, 218, 220, TFT_DGRAY);
  tft.setTextColor(TFT_LGRAY); tft.setTextSize(1);
  tft.setCursor(10, 226); tft.print("Efficiency:");

  int barX = 10, barY = 240, barW = 220, barH = 14;
  tft.fillRoundRect(barX, barY, barW, barH, 4, TFT_DGRAY);
  int fw = (int)((eff / 100.0) * barW);
  if (fw > 0) tft.fillRoundRect(barX, barY, fw, barH, 4, effColor(eff));

  tft.setTextColor(effColor(eff)); tft.setTextSize(1);
  tft.setCursor(barX + barW / 2 - 12, barY + 3);
  tft.print((int)eff); tft.print("%");

  for (int t = 0; t <= 4; t++) {
    int tx = barX + (barW * t / 4);
    tft.drawFastVLine(tx, barY - 4, 4, TFT_LGRAY);
  }

  tft.setTextColor(TFT_DGRAY); tft.setTextSize(1);
  tft.setCursor(10, 264);
  tft.print("Nominal: "); tft.print(NOMINAL_VOLTAGE, 0);
  tft.print("V  |  Next panel in 3s");
}

// ════════════════════════════════════════════════════════════
// LCD — FAULT ICONS
// ════════════════════════════════════════════════════════════
void drawWarningTriangle(int cx, int cy, int sz, uint16_t col) {
  tft.fillTriangle(cx, cy - sz,
                   cx - sz, cy + sz / 2,
                   cx + sz, cy + sz / 2, col);
  tft.fillTriangle(cx,         cy - sz + 5,
                   cx - sz + 4, cy + sz / 2 - 3,
                   cx + sz - 4, cy + sz / 2 - 3, TFT_BG);
  tft.fillRect(cx - 3, cy - sz / 2 + 4, 6, sz / 2, col);
  tft.fillRect(cx - 3, cy + sz / 4,     6, 5,       col);
}

void drawDeadIcon(int cx, int cy, int sz, uint16_t col) {
  for (int t = -2; t <= 2; t++) {
    tft.drawLine(cx - sz + t, cy - sz, cx + sz + t, cy + sz, col);
    tft.drawLine(cx + sz + t, cy - sz, cx - sz + t, cy + sz, col);
  }
  tft.drawCircle(cx, cy, sz + 8, col);
}

void drawHotIcon(int cx, int cy) {
  tft.fillCircle(cx,     cy + 12, 14, TFT_RED);
  tft.fillCircle(cx - 8, cy + 5,  9,  TFT_ORANGE);
  tft.fillCircle(cx + 8, cy + 5,  9,  TFT_ORANGE);
  tft.fillTriangle(cx, cy - 12,
                   cx - 12, cy + 8,
                   cx + 12, cy + 8, TFT_ORANGE);
  tft.fillCircle(cx, cy + 12, 7, TFT_YELLOW);
  tft.fillCircle(cx, cy + 10, 3, TFT_WHITE);
}

void drawDustCloud(int cx, int cy, int drift, uint16_t col) {
  tft.fillCircle(cx + drift,        cy,       15, col);
  tft.fillCircle(cx - 14 + drift,   cy + 6,   11, col);
  tft.fillCircle(cx + 14 + drift,   cy + 6,   11, col);
  tft.fillCircle(cx - 7  + drift,   cy - 9,   10, col);
  tft.fillCircle(cx + 7  + drift,   cy - 9,   10, col);
  tft.fillRect(cx - 24   + drift,   cy + 6,   48, 14, col);
}

// ════════════════════════════════════════════════════════════
// LCD — FAULT SCREEN (holds 5 seconds with countdown)
// ════════════════════════════════════════════════════════════
void showFaultLCD(int panelNum, String faultCode,
                  float voltage, float temp, float hum) {

  uint16_t hdrCol = (faultCode == "F3") ? TFT_RED :
                    (faultCode == "F2") ? TFT_LGRAY : TFT_ORANGE;

  tft.fillScreen(TFT_BG);

  tft.fillRect(0, 0, 240, 30, hdrCol);
  tft.setTextColor(0x0000); tft.setTextSize(1);
  tft.setCursor(8, 8);  tft.print("! FAULT DETECTED - PANEL ");
  tft.print(panelNum);  tft.print(" !");
  tft.setCursor(8, 19); tft.print("Holding display for 5 seconds...");

  int icx = 120, icy = 105;
  if (faultCode == "F1") {
    drawDustCloud(icx, icy, 0, 0xC618);
    tft.setTextColor(TFT_WHITE); tft.setTextSize(2);
    tft.setCursor(46, 148); tft.print("DUST / SAND");

  } else if (faultCode == "F2") {
    drawDeadIcon(icx, icy, 26, TFT_RED);
    tft.drawCircle(icx, icy, 42, TFT_RED);
    tft.setTextColor(TFT_WHITE); tft.setTextSize(2);
    tft.setCursor(40, 148); tft.print("PANEL  DEAD");

  } else if (faultCode == "F3") {
    drawHotIcon(icx, icy);
    drawWarningTriangle(icx, icy - 46, 16, TFT_YELLOW);
    tft.setTextColor(TFT_WHITE); tft.setTextSize(2);
    tft.setCursor(52, 148); tft.print("HOTSPOT!");
  }

  tft.fillRoundRect(10, 170, 220, 106, 8, 0x2104);
  tft.drawRoundRect(10, 170, 220, 106, 8, hdrCol);
  tft.setTextSize(1);

  tft.setTextColor(TFT_LGRAY); tft.setCursor(20, 180);
  tft.print("Fault    : ");
  tft.setTextColor(TFT_RED);
  tft.print(faultCode); tft.print("  "); tft.print(getFaultMsg(faultCode));

  tft.setTextColor(TFT_LGRAY); tft.setCursor(20, 198);
  tft.print("Voltage  : ");
  tft.setTextColor(TFT_CYAN);
  tft.print(voltage, 2); tft.print(" V");

  tft.setTextColor(TFT_LGRAY); tft.setCursor(20, 216);
  tft.print("Temp     : ");
  tft.setTextColor(TFT_ORANGE);
  tft.print(temp, 1); tft.print(" C");

  tft.setTextColor(TFT_LGRAY); tft.setCursor(20, 234);
  tft.print("Humidity : ");
  tft.setTextColor(TFT_CYAN);
  tft.print(hum, 1); tft.print(" %");

  tft.fillRect(0, 308, 240, 12, hdrCol);
  tft.setTextColor(0x0000); tft.setTextSize(1);
  tft.setCursor(46, 311); tft.print("CHECK PANEL NOW!");

  tft.setTextColor(TFT_LGRAY); tft.setTextSize(1);
  tft.setCursor(20, 252); tft.print("Clearing in:");

  int pbX = 20, pbY = 264, pbW = 200, pbH = 8;
  tft.fillRoundRect(pbX, pbY, pbW, pbH, 3, TFT_DGRAY);

  for (int sec = 5; sec >= 1; sec--) {
    tft.fillRect(106, 249, 40, 10, TFT_BG);
    tft.setTextColor(TFT_YELLOW); tft.setTextSize(1);
    tft.setCursor(108, 252);
    tft.print(" "); tft.print(sec); tft.print("s");

    tft.fillRoundRect(pbX, pbY, pbW, pbH, 3, TFT_DGRAY);
    int pbFill = (int)(((float)sec / 5.0) * pbW);
    if (pbFill > 0)
      tft.fillRoundRect(pbX, pbY, pbFill, pbH, 3, hdrCol);

    delay(1000);
  }
}

// ════════════════════════════════════════════════════════════
// SERIAL
// ════════════════════════════════════════════════════════════
void printSerialPanel(int num, float v, float t,
                      float h, String code) {
  Serial.println("--------------------------------------");
  Serial.print("[PANEL "); Serial.print(num); Serial.println("]");
  Serial.print("  Voltage     : "); Serial.print(v, 2);  Serial.println(" V");
  Serial.print("  Temperature : "); Serial.print(t, 1);  Serial.println(" C");
  Serial.print("  Humidity    : "); Serial.print(h, 1);  Serial.println(" %");
  Serial.print("  Status      : ");
  if      (code == "OK") Serial.println("Normal - No Fault");
  else if (code == "F1") Serial.println("[FAULT 1] Low Voltage - Dust/Sand");
  else if (code == "F2") Serial.println("[FAULT 2] Panel Dead - Zero Voltage");
  else if (code == "F3") Serial.println("[FAULT 3] Hotspot - High Temperature");
}

// ════════════════════════════════════════════════════════════
void setup() {
  Serial.begin(115200);

  // ── OLED ─────────────────────────────────────────────────
  Wire.begin(SDA_PIN, SCL_PIN);
  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println("[ERROR] OLED not found!");
    while (true);
  }
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(10, 10); display.println("Solar Fault System");
  display.setCursor(20, 28); display.println("3 Panel Monitor");
  display.setCursor(22, 46); display.println("Initializing...");
  display.display();

  // ── ILI9341 ──────────────────────────────────────────────
  fspi.begin(TFT_CLK, TFT_MISO, TFT_MOSI, TFT_CS);
  tft.begin();
  tft.setRotation(0);
  tft.fillScreen(TFT_BG);
  tft.setTextColor(TFT_GOLD);  tft.setTextSize(2);
  tft.setCursor(18, 110);      tft.print("Solar Monitor");
  tft.setTextColor(TFT_LGRAY); tft.setTextSize(1);
  tft.setCursor(55, 140);      tft.print("Initializing...");

  // ── WiFi + MQTT ──────────────────────────────────────────
  connectWiFi();
  connectMQTT();

  Serial.println("========================================");
  Serial.println("  Solar Fault Detection - 3 Panels");
  Serial.println("  Hardware : ESP32-S3");
  Serial.println("  MQTT     : broker.hivemq.com");
  Serial.println("  Topics   : " TOPIC_BASE "panel1/2/3");
  Serial.println("========================================");

  delay(2000);
  dht1.begin(); dht2.begin(); dht3.begin();
  delay(2000);
  Serial.println("[INFO] All sensors ready.");
}

// ════════════════════════════════════════════════════════════
void loop() {

  // ── Read all sensors ─────────────────────────────────────
  float v1 = readVoltage(VOLTAGE_PIN_1);
  float v2 = readVoltage(VOLTAGE_PIN_2);
  float v3 = readVoltage(VOLTAGE_PIN_3);

  float t1 = readTemp(dht1), t2 = readTemp(dht2), t3 = readTemp(dht3);
  float h1 = readHum(dht1),  h2 = readHum(dht2),  h3 = readHum(dht3);

  String f1 = getFaultCode(v1, t1);
  String f2 = getFaultCode(v2, t2);
  String f3 = getFaultCode(v3, t3);

  float e1 = calcEfficiency(v1);
  float e2 = calcEfficiency(v2);
  float e3 = calcEfficiency(v3);

  // ── Serial output ────────────────────────────────────────
  Serial.println("========================================");
  printSerialPanel(1, v1, t1, h1, f1);
  printSerialPanel(2, v2, t2, h2, f2);
  printSerialPanel(3, v3, t3, h3, f3);

  // ── MQTT publish ─────────────────────────────────────────
  publishAllPanels(v1, t1, h1, f1, e1,
                   v2, t2, h2, f2, e2,
                   v3, t3, h3, f3, e3);

  // ── OLED ─────────────────────────────────────────────────
  showEfficiencyOLED(v1, v2, v3, f1, f2, f3);

  // ── LCD: Panel 1 ─────────────────────────────────────────
  if (f1 != "OK") showFaultLCD(1, f1, v1, t1, h1);
  else { showNormalLCD(1, v1, t1, h1, e1); delay(NORMAL_DISPLAY_MS); }
  showEfficiencyOLED(v1, v2, v3, f1, f2, f3);

  // ── LCD: Panel 2 ─────────────────────────────────────────
  if (f2 != "OK") showFaultLCD(2, f2, v2, t2, h2);
  else { showNormalLCD(2, v2, t2, h2, e2); delay(NORMAL_DISPLAY_MS); }
  showEfficiencyOLED(v1, v2, v3, f1, f2, f3);

  // ── LCD: Panel 3 ─────────────────────────────────────────
  if (f3 != "OK") showFaultLCD(3, f3, v3, t3, h3);
  else { showNormalLCD(3, v3, t3, h3, e3); delay(NORMAL_DISPLAY_MS); }
  showEfficiencyOLED(v1, v2, v3, f1, f2, f3);
}

```
