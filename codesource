// --- 1. LIBRARY DAN DEFINISI PIN ---
#include "FS.h"
#include <WiFi.h>
#include <WebServer.h>
#include <LiquidCrystal_I2C.h> // Library untuk LCD 16x2 I2C
#include <DallasTemperature.h> // Library untuk DS18B20
#include <OneWire.h>
#include <DHT.h>
#include "time.h"
#include <Update.h>
#include <PubSubClient.h>
#include <Wire.h>

// Kredensial WiFi (Ganti dengan kredensial Anda)
char ssid[] = "Neya Neya";
char pass[] = "cakewithmilova";

// Definisikan port server HTTP
WebServer server(80);

// Status koneksi WiFi
bool isWiFiConnected = false;
const char* host = "esp32-sn2s-iot"; // Hostname

// --- NTP Time Configuration ---
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 7 * 3600; // WIB (GMT+7)
const int   daylightOffset_sec = 0;

// --- MQTT Configuration ---
const char* mqtt_server = "broker.emqx.io"; 
const int mqtt_port = 1883; 
const char* mqtt_client_id = "ESP32_SN2S_001"; // HARUS UNIK
const char* mqtt_publish_topic = "sn2s/data/status";

// --- MQTT TOPIK SUBSCRIPTION (BARU) ---
const char* mqtt_subscribe_mode = "sn2s/control/mode";
const char* mqtt_subscribe_led = "sn2s/control/led"; 
const char* mqtt_subscribe_pompa = "sn2s/control/pompa";

const unsigned long mqttPublishInterval = 3000; // Publish setiap 3 detik
unsigned long lastMQTTPublish = 0;

// --- PIN DEKLARASI ---
// Sensor
#define ONE_WIRE_BUS 18     // GPIO 18 untuk DS18B20 (Suhu Media/Air)
#define DHT_PIN 19          // GPIO 19 untuk DHT11 (Suhu & RH Udara Dalam)
#define DHT_TYPE DHT11

// Aktuator (Relay - Active-Low: LOW = ON, HIGH = OFF)
#define POMPA_PIN 25        // GPIO 25 untuk Relay 1 -> Pompa Mini Spray
#define LED_PIN 26          // GPIO 26 untuk Relay 2 -> Lampu LED Tumbuh

// --- INISIALISASI OBJEK ---
LiquidCrystal_I2C lcd(0x27, 16, 2); // LCD I2C (GANTI ALAMAT 0x27 JIKA LCD ANDA BERBEDA)
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);
DHT dht(DHT_PIN, DHT_TYPE);
WiFiClient espClient;
PubSubClient mqttClient(espClient); 

// --- VARIABEL KONTROL & AMBANG BATAS ---
const float SUHU_KRITIS_MAX = 30.0; 
const int RH_KRITIS_MIN = 60;       
const int RH_TARGET_MAX = 62;       

// --- VARIABEL KONTROL POMPA BARU ---
const unsigned long pompaInterval = 4 * 60 * 60 * 1000; // 4 jam dalam milidetik
unsigned long lastPompaCycle = 0;
const unsigned long pompaDurasi = 5 * 1000; // Durasi semprot 5 detik
bool pompaScheduledOn = false;
unsigned long pompaStartTime = 0;

unsigned long lastDS18B20Request = 0;
const unsigned long ds18b20Interval = 2000; 
unsigned long timerSerial = 0;
const unsigned long serialInterval = 5000;
unsigned long lastUpdateMillis = 0; 
unsigned long lastLCDUpdate = 0;
const unsigned long LCD_UPDATE_INTERVAL = 1000;

float suhuMedia = 0.0;
float suhuUdara = 0.0;
int rhUdara = 0;

bool isLedOn = false;
bool isPompaOn = false;

int controlMode = 1; // Mode Kontrol (0=Manual, 1=Auto)
bool manualLedState = false;
bool manualPompaState = false;

// Variabel untuk LCD recovery
int lcdErrorCount = 0;
const int MAX_LCD_ERRORS = 3;

// --- FUNGSI MQTT ---

// Fungsi Callback untuk menerima pesan MQTT (BARU)
void mqttCallback(char* topic, byte* payload, unsigned int length) {
    // Konversi payload menjadi string
    String message;
    for (unsigned int i = 0; i < length; i++) {
        message += (char)payload[i];
    }
    
    Serial.print("MQTT Message arrived [");
    Serial.print(topic);
    Serial.print("]: ");
    Serial.println(message);

    // Handle berdasarkan topik
    if (String(topic) == mqtt_subscribe_mode) {
        // Kontrol Mode: "0" = Manual, "1" = Auto
        if (message == "0" || message == "1") {
            int newMode = message.toInt();
            controlMode = newMode;
            
            // Jika pindah ke AUTO, reset manual state 
            if (controlMode == 1) {
                manualLedState = false;
                manualPompaState = false;
                // Pastikan pompa dimatikan saat pindah ke auto mode
                digitalWrite(POMPA_PIN, HIGH);
                isPompaOn = false;
                pompaScheduledOn = false;
            }
            
            Serial.print("Mode changed via MQTT: ");
            Serial.println(controlMode == 1 ? "AUTO" : "MANUAL");
        }
    }
    else if (String(topic) == mqtt_subscribe_led) {
        // Kontrol LED: "0" = OFF, "1" = ON (hanya di mode Manual)
        if (controlMode == 0 && (message == "0" || message == "1")) {
            manualLedState = (message == "1");
            Serial.print("LED state changed via MQTT: ");
            Serial.println(manualLedState ? "ON" : "OFF");
        } else if (controlMode == 1) {
            Serial.println("LED control ignored: System in AUTO mode");
        }
    }
    else if (String(topic) == mqtt_subscribe_pompa) {
        // Kontrol Pompa: "0" = OFF, "1" = ON (hanya di mode Manual)
        if (controlMode == 0 && (message == "0" || message == "1")) {
            manualPompaState = (message == "1");
            
            // Beri warning jika RH sudah tinggi
            if (manualPompaState && rhUdara >= RH_TARGET_MAX) {
                Serial.println("MQTT WARNING: User memaksa pompa ON meski RH sudah tinggi");
            }
            
            Serial.print("Pompa state changed via MQTT: ");
            Serial.println(manualPompaState ? "ON" : "OFF");
        } else if (controlMode == 1) {
            Serial.println("Pompa control ignored: System in AUTO mode");
        }
    }
}

// Fungsi untuk Reconnect ke Broker MQTT (MODIFIED)
void reconnectMQTT() {
    // Hanya jalankan jika WiFi terhubung
    if (!isWiFiConnected) return; 

    while (!mqttClient.connected()) {
        Serial.print("Attempting MQTT connection...");
        // Coba koneksi
        if (mqttClient.connect(mqtt_client_id)) {
            Serial.println("connected");
            
            // Subscribe ke topik kontrol (BARU)
            if (mqttClient.subscribe(mqtt_subscribe_mode)) {
                Serial.print("Subscribed to: ");
                Serial.println(mqtt_subscribe_mode);
            }
            if (mqttClient.subscribe(mqtt_subscribe_led)) {
                Serial.print("Subscribed to: ");
                Serial.println(mqtt_subscribe_led);
            }
            if (mqttClient.subscribe(mqtt_subscribe_pompa)) {
                Serial.print("Subscribed to: ");
                Serial.println(mqtt_subscribe_pompa);
            }
            
        } else {
            Serial.print("failed, rc=");
            Serial.print(mqttClient.state());
            Serial.println(" try again in 5 seconds");
            delay(5000); 
        }
    }
}

// Handler untuk mengirim status sensor dan aktuator dalam format JSON via MQTT
void publishStatus() {
    if (!mqttClient.connected()) return; 

    // Bangun string JSON
    String json = "{";
    json += "\"suhuMedia\":" + String(suhuMedia, 1) + ",";
    json += "\"suhuUdara\":" + String(suhuUdara, 1) + ",";
    json += "\"rhUdara\":" + String(rhUdara) + ",";
    json += "\"controlMode\":" + String(controlMode) + ",";
    json += "\"isLedOn\":" + String(isLedOn ? 1 : 0) + ",";
    json += "\"isPompaOn\":" + String(isPompaOn ? 1 : 0);
    json += "}";

    // Konversi String ke char array untuk publish
    int len = json.length() + 1;
    char message_buffer[len];
    json.toCharArray(message_buffer, len);
    
    // Publikasikan pesan
    if (mqttClient.publish(mqtt_publish_topic, message_buffer)) {
        Serial.print("MQTT Published to ");
        Serial.print(mqtt_publish_topic);
        Serial.print(": ");
        Serial.println(message_buffer);
    } else {
        Serial.println("MQTT Publish failed.");
    }
    lastMQTTPublish = millis();
}

// --- FUNGSI KONTROL AKTUATOR ---

// Kontrol Pompa Spray (Timer Terjadwal + Emergency RH) - DIPERBAIKI
void kontrolPompa() {
    unsigned long currentMillis = millis();
    
    // Logika Prioritas: Cek Mode Kontrol
    if (controlMode == 0) {
        // MODE MANUAL - Kontrol manual TAPI dengan safety override
        bool shouldPompaBeOn = manualPompaState;
        
        // Safety override: matikan jika RH sudah mencapai target (hanya untuk emergency)
        // TAPI biarkan user tetap bisa menyalakan jika memang ingin
        if (manualPompaState && rhUdara >= RH_TARGET_MAX) {
            // Beri warning tapi biarkan user tetap kontrol
            static unsigned long lastSafetyWarning = 0;
            if (currentMillis - lastSafetyWarning > 10000) { // Warning setiap 10 detik
                Serial.println("SAFETY WARNING: RH sudah mencapai target, pompa sebaiknya dimatikan");
                lastSafetyWarning = currentMillis;
            }
            // Jangan otomatis matikan, biarkan user yang memutuskan
        }
        
        digitalWrite(POMPA_PIN, shouldPompaBeOn ? LOW : HIGH);
        isPompaOn = shouldPompaBeOn;
        pompaScheduledOn = false;
        
        // Debug info
        static unsigned long lastManualDebug = 0;
        if (currentMillis - lastManualDebug > 5000) {
            Serial.printf("MANUAL MODE - Pompa: %s, ManualState: %s, PIN: %s\n", 
                         isPompaOn ? "ON" : "OFF",
                         manualPompaState ? "ON" : "OFF",
                         digitalRead(POMPA_PIN) ? "HIGH" : "LOW");
            lastManualDebug = currentMillis;
        }
        return;
    }
    
    // LOGIKA OTOMATIS - DUAL TRIGGER
    
    // 1. TRIGGER DARURAT: RH di bawah 50% (Prioritas Tertinggi di Auto Mode)
    if (rhUdara != 0 && rhUdara < RH_KRITIS_MIN) {
        if (!isPompaOn) {
            digitalWrite(POMPA_PIN, LOW); // Pompa ON
            isPompaOn = true;
            pompaScheduledOn = false;
            Serial.println("EMERGENCY: Pompa ON - RH rendah!");
        }
        return;
    }
    
    // 2. TRIGGER TERJADUAL: Timer 4 jam
    if (currentMillis - lastPompaCycle >= pompaInterval) {
        if (!pompaScheduledOn && !isPompaOn) {
            pompaScheduledOn = true;
            pompaStartTime = currentMillis;
            digitalWrite(POMPA_PIN, LOW); // Pompa ON
            isPompaOn = true;
            Serial.println("SCHEDULED: Pompa ON - Cycle 4 jam");
        }
    }
    
    // Matikan pompa setelah durasi semprot selesai (5 detik)
    if (pompaScheduledOn && isPompaOn && (currentMillis - pompaStartTime >= pompaDurasi)) {
        digitalWrite(POMPA_PIN, HIGH); // Pompa OFF
        isPompaOn = false;
        pompaScheduledOn = false;
        lastPompaCycle = currentMillis; // Reset timer cycle
        Serial.println("SCHEDULED: Pompa OFF - Durasi 5 detik selesai");
    }
    
    // 3. Safety OFF: Jika RH sudah mencapai target (hanya di auto mode)
    if (isPompaOn && rhUdara >= RH_TARGET_MAX && !pompaScheduledOn) {
        digitalWrite(POMPA_PIN, HIGH); // Pompa OFF
        isPompaOn = false;
        Serial.println("SAFETY: Pompa OFF - RH target tercapai");
    }
}

// Kontrol LED (Fotosintesis dan Manajemen Panas)
void kontrolLED() {
    // Logika 1: Manajemen Panas (Prioritas Tertinggi)
    if (suhuMedia > SUHU_KRITIS_MAX) {
        digitalWrite(LED_PIN, HIGH); // LED OFF (Relay Active-Low)
        isLedOn = false;
        return; // Manajemen Panas selalu memiliki prioritas
    }
    
    // Logika Prioritas: Cek Mode Kontrol setelah Manajemen Panas
    if (controlMode == 0) {
        // MODE MANUAL
        digitalWrite(LED_PIN, manualLedState ? LOW : HIGH);
        isLedOn = manualLedState;
        return; 
    }

    // LOGIKA OTOMATIS 
    struct tm timeinfo;
    // Pastikan waktu NTP sudah sinkron sebelum menjalankan jadwal
    if (getLocalTime(&timeinfo)) { 
        int jamSekarang = timeinfo.tm_hour; 
        
        // 12 jam ON, contoh: ON jam 06:00 - 18:00
        if (jamSekarang >= 6 && jamSekarang < 18) { 
            digitalWrite(LED_PIN, LOW); // LED ON
            isLedOn = true;
        } else {
            digitalWrite(LED_PIN, HIGH); // LED OFF 
            isLedOn = false;
        }
    } else {
        // Jika waktu belum sinkron, default ke OFF (lebih aman)
        digitalWrite(LED_PIN, HIGH);
        isLedOn = false;
    }
}

// --- FUNGSI PEMBACAAN SENSOR ---
void bacaSemuaSensor() {
    // 1. Baca DS18B20 (Suhu Media/Air) - Non-blocking request/read cycle
    if (millis() - lastDS18B20Request >= ds18b20Interval) {
        sensors.requestTemperatures();
        lastDS18B20Request = millis(); // Reset timer setelah request
    }
    float tempC = sensors.getTempCByIndex(0);
    if (tempC != -127.00 && tempC != 85.00) { 
        suhuMedia = tempC;
    }
    
    // 2. Baca DHT11 (Suhu & RH Udara Dalam)
    int rh = dht.readHumidity();
    float temp = dht.readTemperature();

    // Cek kegagalan pembacaan DHT
    if (!isnan(temp) && !isnan(rh)) {
        rhUdara = rh;
        suhuUdara = temp;
        lastUpdateMillis = millis(); 
    }
}

// --- FUNGSI TAMPILAN LOKAL (LCD) dan SERIAL - DIPERBAIKI ---
bool checkLCDConnection() {
    Wire.beginTransmission(0x27); // Ganti dengan address LCD Anda jika berbeda
    byte error = Wire.endTransmission();
    return (error == 0);
}

void initLCD() {
    lcd.init();
    lcd.backlight();
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("SN2S-IoT System");
    lcd.setCursor(0, 1);
    lcd.print("Initializing...");
    delay(1000);
    lcdErrorCount = 0;
    Serial.println("LCD initialized successfully");
}

void tampilLCD() {
    if (millis() - lastLCDUpdate < LCD_UPDATE_INTERVAL) {
        return;
    }
    
    // Cek koneksi LCD
    if (!checkLCDConnection()) {
        lcdErrorCount++;
        Serial.printf("LCD connection error #%d\n", lcdErrorCount);
        
        if (lcdErrorCount >= MAX_LCD_ERRORS) {
            Serial.println("Attempting LCD reinitialization...");
            initLCD(); // Coba inisialisasi ulang
        }
        return;
    }
    
    // Reset error count jika koneksi baik
    lcdErrorCount = 0;

    // Tampilan LCD menggunakan data aktual
    if (!isWiFiConnected) {
        lcd.setCursor(0, 0);
        lcd.print("WiFi GAGAL!     ");
        lcd.setCursor(0, 1);
        lcd.print("Cek Kredensial  ");
    } else {
        lcd.setCursor(0, 0);
        lcd.print("M:");
        lcd.print(suhuMedia, 1); 
        lcd.print("C U:");
        lcd.print(suhuUdara, 1); 
        lcd.print("C "); 

        lcd.setCursor(0, 1);
        lcd.print("RH:");
        lcd.print(rhUdara); 
        lcd.print("% M:");
        lcd.print(controlMode == 1 ? "A" : "M"); 
        
        // Tampilkan status pompa
        lcd.setCursor(12, 1);
        lcd.print(isPompaOn ? "P:ON" : "P:OFF");
    }
    
    lastLCDUpdate = millis();
}

// Fungsi untuk output Serial Monitor
void tampilSerialMonitor() {
    if (millis() - timerSerial >= serialInterval) { 
        Serial.println("========================================");
        if (isWiFiConnected) {
            Serial.print("IP Webserver: "); Serial.println(WiFi.localIP());
            Serial.print("Time: "); 
            struct tm timeinfo;
            if (getLocalTime(&timeinfo)) Serial.println(&timeinfo, "%H:%M:%S %d/%m/%Y");
            else Serial.println("Time not synchronized.");

            Serial.print("Media: "); Serial.print(suhuMedia, 1); Serial.print(" C | ");
            Serial.print("Udara: "); Serial.print(suhuUdara, 1); Serial.print(" C | ");
            Serial.print("RH: "); Serial.print(rhUdara); Serial.println(" %");
            Serial.print("Mode: "); Serial.print(controlMode == 1 ? "AUTO" : "MANUAL");
            Serial.print(" | LED: "); Serial.print(isLedOn ? "ON" : "OFF");
            
            // INFO POMPA YANG DIPERBARUI
            Serial.print(" | Pompa: "); Serial.print(isPompaOn ? "ON" : "OFF");
            if (isPompaOn) {
                if (pompaScheduledOn) {
                    Serial.print(" (Scheduled - ");
                    Serial.print((pompaDurasi - (millis() - pompaStartTime)) / 1000);
                    Serial.print("s remaining)");
                } else if (controlMode == 0) {
                    Serial.print(" (Manual Control)");
                } else {
                    Serial.print(" (Emergency - RH rendah)");
                }
            }
            Serial.println();
            
            if (controlMode == 1) {
                // INFO JADWAL POMPA BERIKUTNYA (hanya di auto mode)
                unsigned long nextCycle = pompaInterval - (millis() - lastPompaCycle);
                Serial.print("Next spray cycle in: ");
                Serial.print(nextCycle / 60000); // Convert to minutes
                Serial.println(" minutes");
            }
            
            // INFO DURASI POMPA
            Serial.print("Pompa schedule: Every ");
            Serial.print(pompaInterval / 3600000); // Convert to hours
            Serial.print(" hours for ");
            Serial.print(pompaDurasi / 1000); // Convert to seconds
            Serial.println(" seconds");
            
            Serial.print("MQTT Status: "); Serial.println(mqttClient.connected() ? "Connected" : "Disconnected");

            if (suhuMedia > SUHU_KRITIS_MAX) {
                Serial.println("!!! PERINGATAN: MANAJEMEN PANAS AKTIF (Media > 30C) !!!");
            }
            
        } else {
            Serial.println("Status: WiFi Disconnected. Web Server Offline.");
        }
        Serial.println("========================================");
        timerSerial = millis();
    }
}

// Fungsi debug sistem
void debugSystem() {
    static unsigned long lastDebug = 0;
    if (millis() - lastDebug > 5000) { // Debug setiap 5 detik
        Serial.println("=== DEBUG SYSTEM INFO ===");
        Serial.printf("Control Mode: %s\n", controlMode == 1 ? "AUTO" : "MANUAL");
        Serial.printf("Pompa State: %s\n", isPompaOn ? "ON" : "OFF");
        Serial.printf("Manual Pompa State: %s\n", manualPompaState ? "ON" : "OFF");
        Serial.printf("Pompa PIN State: %s\n", digitalRead(POMPA_PIN) ? "HIGH" : "LOW");
        Serial.printf("Pompa Scheduled: %s\n", pompaScheduledOn ? "YES" : "NO");
        Serial.printf("RH Udara: %d%%\n", rhUdara);
        Serial.printf("LCD Error Count: %d\n", lcdErrorCount);
        Serial.printf("Free Heap: %d bytes\n", ESP.getFreeHeap());
        
        // Info tambahan untuk mode manual
        if (controlMode == 0) {
            Serial.println("--- MANUAL MODE ACTIVE ---");
            Serial.printf("User Pompa Control: %s\n", manualPompaState ? "ON" : "OFF");
            Serial.printf("Actual Pompa Output: %s\n", digitalRead(POMPA_PIN) ? "OFF" : "ON");
        }
        Serial.println("=========================");
        lastDebug = millis();
    }
}

// --- FUNGSI WEB SERVER DAN KONTROL ---

// HTML untuk antarmuka pengguna
const char* HTML_CONTENT = R"rawliteral(
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SN2S-IoT Control</title>
    <style>
        :root {
            --green-primary: #10B981; /* Emerald Green */
            --green-dark: #059669;
            --gray-light: #F9FAFB;
            --gray-medium: #E5E7EB;
            --white: #ffffff;
            --danger: #EF4444; /* Red */
            --warning: #F59E0B; /* Amber */
        }
        body {
            font-family: 'Inter', sans-serif;
            background-color: var(--gray-light);
            margin: 0;
            padding: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
        }
        .app-container {
            width: 100%;
            max-width: 420px; /* Lebar khas ponsel */
            min-height: 100vh;
            background-color: var(--white);
            box-shadow: 0 0 20px rgba(0, 0, 0, 0.05);
            padding: 20px;
            box-sizing: border-box;
        }
        h1 {
            color: var(--green-dark);
            text-align: center;
            margin-bottom: 25px;
            font-size: 1.6rem;
            border-bottom: 3px solid var(--green-primary);
            padding-bottom: 10px;
        }
        .card {
            background-color: var(--gray-light);
            border-radius: 12px;
            padding: 15px;
            margin-bottom: 20px;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -2px rgba(0, 0, 0, 0.06);
        }
        /* Grid Sensor */
        .sensor-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(100px, 1fr));
            gap: 10px;
        }
        .sensor-item {
            background: var(--white);
            border: 1px solid var(--gray-medium);
            border-radius: 8px;
            padding: 12px;
            text-align: center;
            transition: all 0.3s ease;
        }
        .sensor-label {
            font-size: 0.75rem;
            color: #6B7280;
            font-weight: 500;
            text-transform: uppercase;
        }
        .sensor-value {
            font-size: 1.5rem;
            font-weight: 700;
            color: var(--green-dark);
            margin-top: 5px;
        }

        /* Control Sections */
        h2 {
            font-size: 1.2rem;
            color: #374151;
            margin-top: 0;
            margin-bottom: 10px;
        }
        .control-section {
            display: flex;
            justify-content: space-between;
            align-items: center;
            padding: 10px 0;
            border-bottom: 1px solid var(--gray-medium);
        }
        .control-section:last-child {
            border-bottom: none;
        }
        .control-label {
            font-weight: 600;
            color: #1F2937;
        }
        .disabled-overlay {
            color: #9CA3AF;
            font-weight: 500;
        }

        /* Toggle Switch */
        .switch {
            position: relative;
            display: inline-block;
            width: 50px;
            height: 24px;
        }
        .switch input { opacity: 0; width: 0; height: 0; }
        .slider {
            position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; 
            background-color: #ccc; transition: .4s; border-radius: 24px;
        }
        .slider:before {
            position: absolute; content: ""; height: 18px; width: 18px; left: 3px; bottom: 3px; 
            background-color: white; transition: .4s; border-radius: 50%;
        }
        input:checked + .slider { background-color: var(--green-primary); }
        input:checked + .slider:before { transform: translateX(26px); }
        input:disabled + .slider { opacity: 0.5; cursor: not-allowed; }

        /* Status Badges */
        .status-badge {
            display: inline-block;
            padding: 5px 10px;
            border-radius: 9999px; /* Rounded */
            font-weight: 600;
            font-size: 0.75rem;
            text-transform: uppercase;
        }
        .status-on { background-color: #D1FAE5; color: var(--green-dark); }
        .status-off { background-color: #E5E7EB; color: #6B7280; }
        .status-auto { background-color: #DBEAFE; color: #1D4ED8; }
        .status-manual { background-color: #FEEBC3; color: #D97706; }
        
        /* Alert */
        .alert-danger {
            background-color: #FEE2E2;
            color: var(--danger);
            padding: 10px;
            border-radius: 8px;
            font-weight: 600;
            margin-top: 15px;
            margin-bottom: 20px;
            display: none; 
            border: 1px solid #FCA5A5;
        }
        
        .alert-warning {
            background-color: #FEF3C7;
            color: var(--warning);
            padding: 10px;
            border-radius: 8px;
            font-weight: 600;
            margin-top: 15px;
            margin-bottom: 20px;
            display: none; 
            border: 1px solid #FCD34D;
        }
        
        /* OTA Form Styles */
        .ota-button {
            background-color: #3B82F6; /* Blue */
            color: var(--white);
            padding: 10px 20px;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-size: 1rem;
            font-weight: 600;
            transition: background-color 0.3s ease;
            width: 100%;
            max-width: 300px;
            margin: 10px auto;
            display: block;
        }
        .ota-button:hover {
            background-color: #2563EB;
        }
        
        .footer {
            text-align: center;
            font-size: 0.7rem;
            color: #9CA3AF;
            margin-top: 20px;
        }
        
        .manual-control-info {
            font-size: 0.7rem;
            color: #6B7280;
            margin-top: 5px;
            font-style: italic;
        }
    </style>
</head>
<body>
    <div class="app-container">
        <h1>SN2S-IoT Smart Nursery</h1>

        <div class="alert-danger" id="heatWarning">
            <p style="margin: 0;"><strong style="font-size: 1.1rem;">KRITIS!</strong> Manajemen Panas Aktif! Suhu Media/Air di atas 30.0&deg;C. LED dipaksa OFF.</p>
        </div>

        <div class="alert-warning" id="rhWarning">
            <p style="margin: 0;"><strong>PERINGATAN:</strong> Kelembaban sudah mencapai target. Pompa sebaiknya dimatikan.</p>
        </div>

        <div class="card">
            <h2>Data Mikroklimat</h2>
            <div class="sensor-grid">
                <div class="sensor-item">
                    <div class="sensor-label">Suhu Media</div>
                    <div class="sensor-value" id="suhuMedia">--.- &deg;C</div>
                </div>
                <div class="sensor-item">
                    <div class="sensor-label">Suhu Udara</div>
                    <div class="sensor-value" id="suhuUdara">--.- &deg;C</div>
                </div>
                <div class="sensor-item">
                    <div class="sensor-label">Kelembaban (RH)</div>
                    <div class="sensor-value" id="rhUdara">-- %</div>
                </div>
            </div>
        </div>
        
        <div class="card">
            <h2>Pengaturan Sistem</h2>
            <div class="control-section">
                <span class="control-label">Pilih Mode Kontrol</span>
                <label class="switch">
                    <input type="checkbox" id="modeSwitch" onchange="setControlMode(this.checked)">
                    <span class="slider"></span>
                </label>
                <span id="modeStatus" class="status-badge status-auto">AUTO</span>
            </div>
            
            <h2 style="margin-top: 20px;">Kontrol Aktuator</h2>
            
            <div class="control-section">
                <div>
                    <span class="control-label" id="ledLabel">Lampu LED Tumbuh</span>
                    <div class="manual-control-info" id="ledInfo">Hanya dapat dikontrol di mode Manual</div>
                </div>
                <label class="switch">
                    <input type="checkbox" id="ledSwitch" onchange="setActuator('LED', this.checked)">
                    <span class="slider"></span>
                </label>
                <span id="ledStatus" class="status-badge status-off">OFF</span>
            </div>
            
            <div class="control-section">
                <div>
                    <span class="control-label" id="pompaLabel">Pompa Spray</span>
                    <div class="manual-control-info" id="pompaInfo">Hanya dapat dikontrol di mode Manual</div>
                </div>
                <label class="switch">
                    <input type="checkbox" id="pompaSwitch" onchange="setActuator('Pompa', this.checked)">
                    <span class="slider"></span>
                </label>
                <span id="pompaStatus" class="status-badge status-off">OFF</span>
            </div>
        </div>
        
        <!-- OTA Update Form -->
        <div class="card" style="text-align: center; padding: 15px;">
            <h2>Pembaruan Firmware (OTA)</h2>
            <p style="font-size: 0.8rem; color: #6B7280; margin-bottom: 15px;">Unggah file firmware (.bin) dari Arduino IDE Anda untuk memperbarui ESP32.</p>
            <form method="POST" action="/update" enctype="multipart/form-data" id="otaForm">
                <input type="file" name="firmware" id="firmwareFile" accept=".bin" required 
                       style="margin-bottom: 10px; display: block; width: 100%; padding: 8px; border: 1px solid var(--gray-medium); border-radius: 6px;">
                <button type="submit" id="otaSubmit" class="ota-button">
                    Upload & Update
                </button>
            </form>
            <div id="otaMessage" style="margin-top: 10px; font-weight: 600; color: #374151;"></div>
        </div>

        <div class="footer">
            <span id="lastUpdate">Update Terakhir: --:--:--</span><br>
            <span style="font-size: 0.65rem;">Host: %s</span>
        </div>
    </div>
    
    <script>
        const POLLING_INTERVAL = 3000; 
        const CRITICAL_TEMP = 30.0;
        const RH_MAX_TARGET = 52;
        
        // Helper untuk memformat waktu
        function formatTime(ms) {
            const date = new Date(ms);
            const hours = date.getHours().toString().padStart(2, '0');
            const minutes = date.getMinutes().toString().padStart(2, '0');
            const seconds = date.getSeconds().toString().padStart(2, '0');
            return `${hours}:${minutes}:${seconds}`;
        }

        // Fungsi untuk mengambil status data dari ESP32
        async function fetchStatus() {
            try {
                const response = await fetch('/status');
                if (!response.ok) throw new Error('Network response was not ok');
                const data = await response.json();
                
                // Waktu update
                document.getElementById('lastUpdate').textContent = 'Update Terakhir: ' + formatTime(Date.now());
                
                // 1. Update Sensor Data
                document.getElementById('suhuMedia').textContent = data.suhuMedia.toFixed(1) + ' °C';
                document.getElementById('suhuUdara').textContent = data.suhuUdara.toFixed(1) + ' °C';
                document.getElementById('rhUdara').textContent = data.rhUdara + ' %';

                // 2. Update Mode Status
                const isAuto = data.controlMode === 1;
                const modeSwitch = document.getElementById('modeSwitch');
                const modeStatus = document.getElementById('modeStatus');
                modeSwitch.checked = isAuto;
                modeStatus.textContent = isAuto ? 'AUTO' : 'MANUAL';
                modeStatus.className = 'status-badge ' + (isAuto ? 'status-auto' : 'status-manual');

                // 3. Update LED Status
                const ledSwitch = document.getElementById('ledSwitch');
                const ledStatus = document.getElementById('ledStatus');
                const ledLabel = document.getElementById('ledLabel');
                const ledInfo = document.getElementById('ledInfo');
                
                ledSwitch.checked = data.isLedOn;
                ledStatus.textContent = data.isLedOn ? 'ON' : 'OFF';
                ledStatus.className = 'status-badge ' + (data.isLedOn ? 'status-on' : 'status-off');
                ledSwitch.disabled = isAuto; 
                ledLabel.className = isAuto ? 'disabled-overlay' : 'control-label';
                ledInfo.textContent = isAuto ? 'Kontrol dinonaktifkan di mode Auto' : 'Kontrol manual aktif';

                // 4. Update Pompa Status
                const pompaSwitch = document.getElementById('pompaSwitch');
                const pompaStatus = document.getElementById('pompaStatus');
                const pompaLabel = document.getElementById('pompaLabel');
                const pompaInfo = document.getElementById('pompaInfo');
                
                pompaSwitch.checked = data.isPompaOn;
                pompaStatus.textContent = data.isPompaOn ? 'ON' : 'OFF';
                pompaStatus.className = 'status-badge ' + (data.isPompaOn ? 'status-on' : 'status-off');
                pompaSwitch.disabled = isAuto; 
                pompaLabel.className = isAuto ? 'disabled-overlay' : 'control-label';
                pompaInfo.textContent = isAuto ? 'Kontrol dinonaktifkan di mode Auto' : 'Kontrol manual aktif';
                
                // Update label pompa berdasarkan mode
                if (isAuto && data.isPompaOn) {
                    pompaLabel.textContent = `Pompa Spray (Aktif: Target RH < ${RH_MAX_TARGET}%)`;
                    pompaLabel.className = 'control-label'; 
                } else {
                    pompaLabel.textContent = 'Pompa Spray';
                    pompaLabel.className = isAuto ? 'disabled-overlay' : 'control-label';
                }

                // 5. Update Peringatan Panas
                const heatWarning = document.getElementById('heatWarning');
                if (data.suhuMedia > CRITICAL_TEMP) {
                    heatWarning.style.display = 'block';
                } else {
                    heatWarning.style.display = 'none';
                }
                
                // 6. Update Peringatan RH
                const rhWarning = document.getElementById('rhWarning');
                if (data.rhUdara >= RH_MAX_TARGET && !isAuto) {
                    rhWarning.style.display = 'block';
                } else {
                    rhWarning.style.display = 'none';
                }

            } catch (error) {
                console.error('Fetch error:', error);
            }
        }

        // Fungsi untuk mengirim perintah Mode
        function setControlMode(isAuto) {
            const mode = isAuto ? 1 : 0;
            const originalState = document.getElementById('modeSwitch').checked;
            
            fetch(`/setMode?value=${mode}`)
                .then(response => {
                    if (!response.ok) throw new Error('Gagal mengganti mode');
                    fetchStatus();
                })
                .catch(error => {
                    console.error(error);
                    // Kembalikan posisi switch jika gagal
                    document.getElementById('modeSwitch').checked = originalState;
                });
        }

        // Fungsi untuk mengirim perintah Aktuator (LED/Pompa)
        function setActuator(actuator, isOn) {
            const value = isOn ? 1 : 0;
            const endpoint = actuator === 'LED' ? '/setLED' : '/setPompa';
            const switchId = actuator.toLowerCase() + 'Switch';
            const switchElement = document.getElementById(switchId);
            
            const previousState = !isOn; 
            
            if (document.getElementById('modeSwitch').checked) {
                console.warn('Aktuator hanya bisa dikontrol di mode MANUAL.');
                switchElement.checked = previousState; 
                return;
            }

            switchElement.disabled = true;

            fetch(`${endpoint}?value=${value}`)
                .then(response => {
                    if (!response.ok) throw new Error('Gagal mengontrol ' + actuator);
                    fetchStatus();
                })
                .catch(error => {
                    console.error(error);
                    switchElement.checked = previousState; 
                })
                .finally(() => {
                    if (document.getElementById('modeSwitch').checked === false) {
                        switchElement.disabled = false;
                    }
                });
        }

        // Tambahkan event listener untuk form OTA
        document.getElementById('otaForm').addEventListener('submit', function(e) {
            const fileInput = document.getElementById('firmwareFile');
            const messageBox = document.getElementById('otaMessage');
            const submitButton = document.getElementById('otaSubmit');
            
            if (fileInput.files.length === 0) {
                messageBox.style.color = 'var(--danger)';
                messageBox.textContent = 'Pilih file .bin firmware terlebih dahulu.';
                e.preventDefault();
                return;
            }

            messageBox.style.color = '#D97706';
            messageBox.textContent = 'Memulai upload... JANGAN matikan daya. Ini mungkin butuh waktu.';
            submitButton.disabled = true;
        });

        // Mulai polling saat halaman dimuat
        document.addEventListener('DOMContentLoaded', () => {
            fetchStatus(); 
            setInterval(fetchStatus, POLLING_INTERVAL); 
        });
    </script>
</body>
</html>
)rawliteral";


// Handler untuk menyajikan halaman utama (HTML)
void handleRoot() {
    if (!isWiFiConnected) {
        server.send(503, "text/plain", "Service Unavailable: Not connected to WiFi.");
        return;
    }
    // Ganti placeholder di HTML content
    String html = HTML_CONTENT;
    html.replace("%s", WiFi.localIP().toString().c_str());
    server.send(200, "text/html", html);
}

// Handler untuk mengirim status sensor dan aktuator dalam format JSON
void handleStatus() {
    if (!isWiFiConnected) {
        server.send(503, "text/plain", "Service Unavailable: Not connected to WiFi.");
        return;
    }
    String json = "{";
    json += "\"suhuMedia\":" + String(suhuMedia, 1) + ",";
    json += "\"suhuUdara\":" + String(suhuUdara, 1) + ",";
    json += "\"rhUdara\":" + String(rhUdara) + ",";
    json += "\"controlMode\":" + String(controlMode) + ",";
    json += "\"isLedOn\":" + String(isLedOn ? 1 : 0) + ",";
    json += "\"isPompaOn\":" + String(isPompaOn ? 1 : 0);
    json += "}";
    server.send(200, "application/json", json);
}

// Handler untuk mengubah Mode Kontrol (/setMode?value=X)
void handleSetMode() {
    if (!isWiFiConnected) {
        server.send(503, "text/plain", "Service Unavailable: Not connected to WiFi.");
        return;
    }
    if (server.hasArg("value")) {
        int newMode = server.arg("value").toInt();
        if (newMode == 0 || newMode == 1) {
            controlMode = newMode;
            // Jika pindah ke AUTO, reset manual state 
            if (controlMode == 1) {
                manualLedState = false;
                manualPompaState = false;
                // Pastikan pompa dimatikan saat pindah ke auto mode
                digitalWrite(POMPA_PIN, HIGH);
                isPompaOn = false;
                pompaScheduledOn = false;
            }
            server.send(200, "text/plain", "OK");
            Serial.printf("Mode changed to: %s\n", controlMode == 1 ? "AUTO" : "MANUAL");
            return;
        }
    }
    server.send(400, "text/plain", "Invalid Value");
}

// Handler untuk mengubah status LED Manual (/setLED?value=X)
void handleSetLED() {
    if (!isWiFiConnected) {
        server.send(503, "text/plain", "Service Unavailable: Not connected to WiFi.");
        return;
    }
    // Hanya izinkan kontrol manual jika mode = 0
    if (controlMode == 0 && server.hasArg("value")) {
        manualLedState = server.arg("value").toInt() == 1;
        server.send(200, "text/plain", "OK");
        Serial.printf("LED manual di-set ke: %s\n", manualLedState ? "ON" : "OFF");
        return;
    }
    server.send(403, "text/plain", "Forbidden: Not in Manual Mode or Invalid Value");
}

// Handler untuk mengubah status Pompa Manual (/setPompa?value=X) - DIPERBAIKI
void handleSetPompa() {
    if (!isWiFiConnected) {
        server.send(503, "text/plain", "Service Unavailable: Not connected to WiFi.");
        return;
    }
    
    // Hanya izinkan kontrol manual jika mode = 0
    if (controlMode == 0 && server.hasArg("value")) {
        int newValue = server.arg("value").toInt();
        if (newValue == 0 || newValue == 1) {
            manualPompaState = (newValue == 1);
            
            // Beri warning jika RH sudah tinggi tapi tetap izinkan
            if (manualPompaState && rhUdara >= RH_TARGET_MAX) {
                Serial.println("WARNING: User memaksa pompa ON meski RH sudah tinggi");
            }
            
            server.send(200, "text/plain", "OK");
            Serial.printf("Pompa manual di-set ke: %s\n", manualPompaState ? "ON" : "OFF");
            return;
        }
    }
    server.send(403, "text/plain", "Forbidden: Not in Manual Mode or Invalid Value");
}

// Handler untuk pembaruan OTA Web (/update)
void handleWebUpdate() {
    // Handler untuk POST request ke /update. File upload ditangani di dalam fungsi ini.
    server.on("/update", HTTP_POST, []() {
        // Callback setelah semua data diunggah dan pembaruan selesai/gagal
        server.sendHeader("Connection", "close");
        if (Update.hasError()) {
            Serial.println("Update Gagal");
            // Mencetak error ke Serial Monitor agar dapat didiagnosis.
            Update.printError(Serial);
            server.send(500, "text/html", "<h2>Update Gagal!</h2><p>Pastikan file firmware (.bin) valid. <a href='/'>Kembali</a></p>");
        } else {
            Serial.println("Update Sukses. Rebooting...");
            // Pemberitahuan sukses di browser, dan refresh setelah 7 detik
            server.send(200, "text/html", "<h2>Update Sukses!</h2><p>ESP32 akan Reboot dalam 7 detik. <meta http-equiv='refresh' content='7; url=/' /></p>");
            // Panggil fungsi restart setelah mengirim respons
            delay(100);
            ESP.restart();
        }
    }, []() {
        // Handler yang dipanggil selama proses upload file
        HTTPUpload& upload = server.upload();
        if (upload.status == UPLOAD_FILE_START) {
            Serial.printf("Pembaruan dimulai: %s\n", upload.filename.c_str());
            
            // Periksa ukuran partisi dan mulai proses update
            if (!Update.begin(UPDATE_SIZE_UNKNOWN)) { 
                Update.printError(Serial);
            }
        } else if (upload.status == UPLOAD_FILE_WRITE) {
            // Tulis data ke flash
            if (Update.write(upload.buf, upload.currentSize) != upload.currentSize) {
                Serial.println("Write failed");
            }
        } else if (upload.status == UPLOAD_FILE_END) {
            if (Update.end(true)) { // true = commit
                Serial.printf("Pembaruan selesai: %u bytes\n", upload.totalSize);
            } else {
                Update.printError(Serial);
            }
        }
    });
}

// --- FUNGSI SETUP (DIMODIFIKASI) ---
void setup() {
    Serial.begin(115200);
    Serial.println("\n[SN2S-IoT] Inisialisasi...");

    // Inisialisasi I2C dengan konfigurasi yang lebih robust
    Wire.begin();
    Wire.setClock(100000); // Set I2C clock ke 100kHz (lebih stabil)
    
    // 1. Setup Pin Aktuator (Default HIGH = OFF)
    pinMode(POMPA_PIN, OUTPUT);
    pinMode(LED_PIN, OUTPUT);
    digitalWrite(POMPA_PIN, HIGH);
    digitalWrite(LED_PIN, HIGH);
    Serial.println("Aktuator siap.");

    // 2. Inisialisasi Sensor dan LCD
    initLCD(); // Gunakan fungsi initLCD yang baru

    sensors.begin();
    dht.begin();
    Serial.println("Sensor dan LCD siap.");

    // 3. Koneksi WiFi
    WiFi.mode(WIFI_STA); // Mode Station
    WiFi.begin(ssid, pass);
    Serial.printf("Menghubungkan ke WiFi '%s'", ssid);
    
    int timeout = 0;
    while (WiFi.status() != WL_CONNECTED && timeout < 20) {
        delay(500);
        Serial.print(".");
        lcd.setCursor(8 + (timeout % 8), 1);
        lcd.print(".");
        timeout++;
    }

    if (WiFi.status() == WL_CONNECTED) {
        isWiFiConnected = true;
        Serial.println("\nWiFi Terhubung!");
        Serial.print("IP Address: ");
        Serial.println(WiFi.localIP());
        lcd.clear();
        lcd.print("IP:");
        lcd.print(WiFi.localIP());
        
        // 4. Inisialisasi Web Server dan Routing
        server.on("/", HTTP_GET, handleRoot);
        server.on("/status", HTTP_GET, handleStatus);
        server.on("/setMode", HTTP_GET, handleSetMode);
        server.on("/setLED", HTTP_GET, handleSetLED);
        server.on("/setPompa", HTTP_GET, handleSetPompa);
        handleWebUpdate(); // Mengaktifkan handler POST untuk /update
        server.begin();
        Serial.println("Web Server dimulai.");

        // 5. Sinkronisasi Waktu NTP
        configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
        Serial.println("NTP Time Sync dimulai.");
        
        // 6. Inisialisasi MQTT (DIMODIFIKASI)
        mqttClient.setServer(mqtt_server, mqtt_port);
        mqttClient.setCallback(mqttCallback); // Set callback function untuk menerima pesan
        Serial.println("MQTT Klien diset.");

    } else {
        isWiFiConnected = false;
        Serial.println("\nGagal terhubung ke WiFi.");
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("WiFi GAGAL!");
        lcd.setCursor(0, 1);
        lcd.print("Cek Kredensial");
    }
}

// --- FUNGSI LOOP (DIMODIFIKASI) ---
void loop() {
    // 1. Cek dan Jaga Koneksi WiFi
    if (WiFi.status() != WL_CONNECTED) {
        if (isWiFiConnected) {
            Serial.println("WiFi terputus. Mencoba menghubungkan kembali...");
            isWiFiConnected = false;
        }
        WiFi.reconnect();
        // Batasi operasi lain jika WiFi mati
        return;
    }
    // Jika baru saja terhubung kembali
    if (!isWiFiConnected && WiFi.status() == WL_CONNECTED) {
      isWiFiConnected = true;
      Serial.println("WiFi terhubung kembali.");
    }

    // 2. Handle Client dan MQTT Loop
    server.handleClient();
    mqttClient.loop();

    // 3. Cek Koneksi MQTT dan Publish Status
    if (!mqttClient.connected()) {
        reconnectMQTT();
    } else if (millis() - lastMQTTPublish >= mqttPublishInterval) {
        publishStatus();
    }
    
    // 4. Pembacaan Sensor, Kontrol, dan Output Periodik
    bacaSemuaSensor();
    kontrolPompa();
    kontrolLED();
    tampilLCD();
    tampilSerialMonitor();
    debugSystem(); // Tambahkan debug system
}
