#include <Servo.h>
#include "HX711.h"
#include <RTClib.h>
#include <Wire.h>

#define IR_SENSOR_PIN 2
#define SERVO_PIN 9
#define BUZZER_PIN 6
#define LED_PIN 7
#define HX711_DOUT A1
#define HX711_CLK A0

HX711 scale;
RTC_DS3231 rtc;
Servo feederServo;

// Feeding control
unsigned long lastFeedTime = 0;
const unsigned long FEED_COOLDOWN = 60UL * 60UL * 1000UL; // 1 hour
int feedCountThisHour = 0;
const int FEED_LIMIT = 3;

// State
bool aiConfirmedBird = false;

void setup() {
  Serial.begin(9600);
  feederServo.attach(SERVO_PIN);
  pinMode(IR_SENSOR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);

  scale.begin(HX711_DOUT, HX711_CLK);
  scale.set_scale(2280.f);
  scale.tare();

  if (!rtc.begin()) {
    Serial.println("Couldn't find RTC!");
    while (1);
  }

  Serial.println("Smart Bird Feeder (Advanced) Ready.");
}

void loop() {
  DateTime now = rtc.now();
  float foodWeight = scale.get_units();
  int irStatus = digitalRead(IR_SENSOR_PIN);

  // Serial read from Pi
  if (Serial.available()) {
    char input = Serial.read();
    if (input == '1') {
      aiConfirmedBird = true;
      Serial.println("AI confirmation received.");
    }
  }

  // Reset feed count every hour
  if (now.minute() == 0 && now.second() == 0 && (millis() - lastFeedTime) > 5000) {
    feedCountThisHour = 0;
    Serial.println("Feed count reset for new hour.");
  }

  if (irStatus == LOW && aiConfirmedBird && millis() - lastFeedTime > FEED_COOLDOWN && feedCountThisHour < FEED_LIMIT) {
    dispenseFood(now);
    aiConfirmedBird = false;
    lastFeedTime = millis();
    feedCountThisHour++;
  }

  delay(300); // Reduce CPU usage
}

void dispenseFood(DateTime timestamp) {
  Serial.print("Dispensing food at: ");
  Serial.print(timestamp.hour()); Serial.print(":");
  Serial.print(timestamp.minute()); Serial.print(":");
  Serial.println(timestamp.second());

  digitalWrite(LED_PIN, HIGH);
  tone(BUZZER_PIN, 1000, 300);
  feederServo.write(90); delay(2000); feederServo.write(0);
  digitalWrite(LED_PIN, LOW);

  Serial.print("Current feed count: ");
  Serial.println(feedCountThisHour + 1);
}
