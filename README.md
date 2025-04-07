#include <HX711.h>
#include <Servo.h>
#include <ESP32CAM.h>  // Include camera library for ESP32-CAM

// Pin definitions
#define IR_PIN 7
#define SERVO_PIN 9
#define LOAD_CELL_DT A1
#define LOAD_CELL_SCK A0

// Set up components
HX711 scale;
Servo foodServo;
ESP32CAM camera;

// Set up variables
long weightThreshold = 1000;  // Weight threshold for dispensing food (example)
bool birdDetected = false;

void setup() {
  Serial.begin(115200);

  // Set up the load cell
  scale.begin(LOAD_CELL_DT, LOAD_CELL_SCK);
  scale.set_scale(2280); // Adjust calibration factor as needed
  scale.tare(); // Set zero offset

  // Set up servo motor
  foodServo.attach(SERVO_PIN);
  
  // Set up the IR sensor
  pinMode(IR_PIN, INPUT);
  
  // Set up the ESP32-CAM
  camera.begin();
  camera.setResolution(ESP32CAM::Resolution::UXGA);
  
  Serial.println("System Ready");
}

void loop() {
  if (digitalRead(IR_PIN) == HIGH) {  // Bird detected
    birdDetected = true;
    long weight = scale.get_units(10); // Read weight from load cell
    Serial.print("Weight: ");
    Serial.println(weight);

    if (birdDetected && weight >= weightThreshold) {
      // Capture image from the ESP32-CAM
      camera.capture();
      Serial.println("Image Captured");

      // Save the image or upload it to cloud (this part depends on your setup)
      
      // Dispense food
      foodServo.write(90);  // Activate food dispensing (adjust angle as needed)
      delay(5000);  // Keep the servo activated for 5 seconds to dispense food
      foodServo.write(0);  // Stop dispensing food

      birdDetected = false;  // Reset bird detection
    }
  }

  delay(1000); // Delay for a while before next reading
}
