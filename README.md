/*
  Soil-Moisture Irrigation Rover (Arduino UNO)
  - Uses HW-080 capacitive soil sensor
  - SG90 servo dips sensor into soil
  - 5V DC pump controlled via relay
  - Moves rover forward step by step
*/

#include <Servo.h>

// ---------------- Pins ----------------
const uint8_t ENA = 5;   // Left motor PWM
const uint8_t IN1 = 11;
const uint8_t IN2 = 10;
const uint8_t IN3 = 9;
const uint8_t IN4 = 8;
const uint8_t ENB = 6;   // Right motor PWM

const uint8_t RELAY_PIN = 7;   // Pump Relay (Active LOW)

const uint8_t SERVO_PIN = 3;   // SG90 servo pin
const uint8_t SOIL_PIN  = A0;  // HW-080 capacitive soil sensor

// ---------------- Calibration ----------------
// Measure your own values with Serial Monitor
int wetADC = 350;    // ADC in wet soil
int dryADC = 1023;   // ADC in dry soil
int DRY_THRESHOLD = 60; // % dryness to trigger watering

// Motion timing
uint16_t stepForwardMs = 1200; 
uint8_t  motorPWM      = 180;  

// Watering time
uint16_t waterMsMin = 2500;  
uint16_t waterMsMax = 8000;  

// Servo
uint8_t servoUp   = 10;  
uint8_t servoDown = 95;  
uint16_t settleMs = 1500; 

Servo soilServo;

// -------------- Motor Helpers ----------------
void motorsStop() {
  digitalWrite(IN1, LOW); digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW); digitalWrite(IN4, LOW);
  analogWrite(ENA, 0); analogWrite(ENB, 0);
}

void motorsForward(uint8_t pwm, uint16_t ms) {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
  analogWrite(ENA, pwm); analogWrite(ENB, pwm);
  delay(ms);
  motorsStop();
}

// -------------- Soil Sensor Helpers ----------------
int adcToDryness(int adc) {
  long p = map(adc, wetADC, dryADC, 0, 100);
  if (p < 0) p = 0;
  if (p > 100) p = 100;
  return (int)p;
}

// Read soil dryness (0-100%)
int readSoilDryness() {
  soilServo.write(servoDown);
  delay(settleMs);

  long sum = 0;
  for (int i = 0; i < 5; i++) { 
    sum += analogRead(SOIL_PIN); 
    delay(50); 
  }
  int adc = sum / 5;

  soilServo.write(servoUp);

  int dryness = adcToDryness(adc);
  Serial.print(F("Soil ADC=")); Serial.print(adc);
  Serial.print(F(" Dryness=")); Serial.print(dryness); Serial.println(F("%"));
  return dryness;
}

// Pump control (Active LOW relay)
void pumpWater(uint16_t ms) {
  Serial.print(F("Pump ON for ")); Serial.print(ms); Serial.println(F(" ms"));
  digitalWrite(RELAY_PIN, LOW);   // Relay ON → Pump ON
  delay(ms);
  digitalWrite(RELAY_PIN, HIGH);  // Relay OFF → Pump OFF
  Serial.println(F("Pump OFF"));
}

// Map dryness to watering time
uint16_t computeWaterMs(int dryness) {
  if (dryness < DRY_THRESHOLD) return 0;
  long ms = map(dryness, DRY_THRESHOLD, 100, waterMsMin, waterMsMax);
  if (ms < waterMsMin) ms = waterMsMin;
  if (ms > waterMsMax) ms = waterMsMax;
  return (uint16_t)ms;
}

// ---------------- Setup ----------------
void setup() {
  Serial.begin(115200);

  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  pinMode(ENA, OUTPUT); pinMode(ENB, OUTPUT);
  motorsStop();

  pinMode(RELAY_PIN, OUTPUT); 
  digitalWrite(RELAY_PIN, HIGH);  // Keep pump OFF initially

  soilServo.attach(SERVO_PIN);
  soilServo.write(servoUp);

  delay(500);
  Serial.println(F("Irrigation Rover ready."));
}

// ---------------- Loop ----------------
void loop() {
  Serial.println(F("Moving forward..."));
  motorsForward(motorPWM, stepForwardMs);
  delay(2000);

  int dryness = readSoilDryness();

  uint16_t wms = computeWaterMs(dryness);
  if (wms > 0) {
    Serial.print(F("Soil dry → Watering ")); Serial.print(wms); Serial.println(F(" ms"));
    pumpWater(wms);
  } else {
    Serial.println(F("Soil OK → No watering"));
  }

  delay(1200);
}
