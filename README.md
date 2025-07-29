#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <SparkFun_MAX3010x.h>
#include <Adafruit_BMP085.h>
#include <DHT.h>
#include <Servo.h>

// --- OLED ---
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// --- Pulse Oximeter ---
MAX30105 pulseOximeter;

// --- DHT11 Sensor ---
#define DHTPIN 2
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

// --- BMP180 Sensor ---
Adafruit_BMP085 bmp;

// --- Servo Pins ---
Servo servos[5];
int servoPins[5] = {5, 6, 9, 10, 11};

// --- Vibration Motor Pins ---
#define ENA 3
#define IN1 7
#define IN2 8

// --- Time Durations ---
#define SYSTEM1_DURATION 90000 // 90 seconds
#define SYSTEM2_DURATION 60000
#define VIBRATION_ON_TIME 5000
#define VIBRATION_OFF_TIME 2000

unsigned long modeStartTime;
unsigned long system2StartTime = 0;
unsigned long vibrationMillis;
bool vibrationState = false;

void setup() {
  Serial.begin(115200);

  // --- OLED Init ---
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("SSD1306 allocation failed"));
    while (1);
  }

  // --- Pulse Oximeter Init ---
  if (!pulseOximeter.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30105 not found");
    while (1);
  }
  pulseOximeter.setup();

  // --- DHT & BMP180 Init ---
  dht.begin();
  if (!bmp.begin()) {
    Serial.println("BMP180 not found");
    while (1);
  }

  // --- Servo Init ---
  for (int i = 0; i < 5; i++) {
    servos[i].attach(servoPins[i]);
    servos[i].write(90);
  }

  // --- Vibration Motor Init ---
  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);

  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println("Setup complete");
  display.display();

  modeStartTime = millis();
}

void loop() {
  unsigned long currentMillis = millis();

  if (currentMillis - modeStartTime < SYSTEM1_DURATION) {
    runSystem1(currentMillis);
  } else {
    if (system2StartTime == 0) system2StartTime = currentMillis;
    runSystem2(currentMillis);
  }
}

// ==================== SYSTEM 1 ====================
void runSystem1(unsigned long currentMillis) {
  unsigned long elapsedTime = currentMillis - modeStartTime;

  display.clearDisplay();
  display.setTextSize(1);

  if (elapsedTime < 30000) {
    // --- Mode 1: Pulse Monitor ---
    display.setCursor(0, 0);
    display.println("Mode 1: Pulse");

    long irValue = pulseOximeter.getIR();
    if (checkForPulse(irValue)) {
      float heartRate = pulseOximeter.getHeartRate();
      float SpO2 = pulseOximeter.getSpO2();

      display.print("HR: "); display.println(heartRate);
      display.print("SpO2: "); display.println(SpO2);

      if (SpO2 < 95) { // Trigger vibration motor
        digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
      } else {
        digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
      }
    } else {
      display.println("No pulse detected");
    }

  } else if (elapsedTime < 60000) {
    // --- Mode 2: All Servos Move Continuously ---
    display.setCursor(0, 0);
    display.println("Mode 2: All Fingers");
    moveAllServosContinuously();
  } else {
    // --- Mode 3: Each Servo One by One ---
    display.setCursor(0, 0);
    display.println("Mode 3: One by One");
    moveServosOneByOne();
  }

  display.display();
  delay(500);
}

boolean checkForPulse(long irValue) {
  return irValue > 100000;
}

// ==================== SYSTEM 2 ====================
void runSystem2(unsigned long currentMillis) {
  if (currentMillis - system2StartTime > SYSTEM2_DURATION) {
    system2StartTime = 0;
    modeStartTime = millis();
    return;
  }

  if (currentMillis - vibrationMillis >= (vibrationState ? VIBRATION_ON_TIME : VIBRATION_OFF_TIME)) {
    vibrationState = !vibrationState;
    digitalWrite(IN1, vibrationState ? HIGH : LOW);
    digitalWrite(IN2, LOW);
    vibrationMillis = currentMillis;
  }

  displaySensorReadings();
}

void displaySensorReadings() {
  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(0, 0);

  float temp = bmp.readTemperature();
  float pressure = bmp.readPressure() / 100.0;
  float humidity = dht.readHumidity();

  display.print("Temp: "); display.print(temp); display.println(" C");
  display.print("Pres: "); display.print(pressure); display.println(" hPa");
  display.print("Hum:  "); display.print(humidity); display.println(" %");
  display.display();

  delay(2000);
}

// ==================== SERVO FUNCTIONS ====================
void moveAllServosContinuously() {
  for (int angle = 0; angle <= 180; angle += 10) {
    for (int i = 0; i < 5; i++) servos[i].write(angle);
    delay(100);
  }
  for (int angle = 180; angle >= 0; angle -= 10) {
    for (int i = 0; i < 5; i++) servos[i].write(angle);
    delay(100);
  }
}

void moveServosOneByOne() {
  for (int i = 0; i < 5; i++) {
    for (int angle = 0; angle <= 180; angle += 10) {
      servos[i].write(angle);
      delay(50);
    }
    for (int angle = 180; angle >= 0; angle -= 10) {
      servos[i].write(angle);
      delay(50);
    }
  }
}
