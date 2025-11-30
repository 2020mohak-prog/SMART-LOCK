#include <LiquidCrystal_I2C.h>
#include <Wire.h>
#include <RTClib.h>
#include <Servo.h>

// LCD Setup
LiquidCrystal_I2C lcd(0x27, 16, 2);

// RTC Setup
RTC_DS3231 rtc;

// Servo Setup
Servo lockServo;

// Pin Definitions
const int PIR_PIN = 2;
const int BUZZER_PIN = 3;
const int SERVO_PIN = 9;
const int JOY_SW_PIN = 5;
const int JOY_X_PIN = A0;
const int JOY_Y_PIN = A1;

// System Variables
int systemState = 0;  // 0=Idle, 1=Active, 2=Verify, 3=Open, 4=Alert
String enteredCode = "";
String correctCode = "UUDDLR";  // 6 character code
unsigned long unlockTime = 0;
const unsigned long UNLOCK_DURATION = 10000;  // 10 seconds

// Joystick Thresholds
const int JOY_THRESHOLD_LOW = 400;
const int JOY_THRESHOLD_HIGH = 600;

// Timing variables
unsigned long lastPIRDetection = 0;
unsigned long lastDisplayUpdate = 0;
unsigned long stateActiveStart = 0;
const unsigned long PIR_COOLDOWN = 2000;
const unsigned long DISPLAY_UPDATE_INTERVAL = 1000;

// Security variables
int failedAttempts = 0;
const int MAX_FAILED_ATTEMPTS = 3;

void setup() {
  Serial.begin(9600);
  Serial.println("=== Safe-Hub Security System ===");
  
  // Initialize pins
  pinMode(PIR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(JOY_SW_PIN, INPUT_PULLUP);
  
  // Initialize I2C
  Wire.begin();
  
  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.clear();
  
  // Initialize RTC
  if (rtc.begin()) {
    Serial.println("RTC DS3231 initialized");
    if (rtc.lostPower()) {
      rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
      Serial.println("RTC time set from compile time");
    }
  } else {
    Serial.println("ERROR: RTC not found!");
    // Continue without RTC
  }
  
  // Initialize Servo
  lockServo.attach(SERVO_PIN);
  lockDrawer();
  
  // Welcome sequence
  welcomeSequence();
  
  Serial.println("System ready - Waiting for motion...");
}

void loop() {
  // Update display with live time every second
  if (millis() - lastDisplayUpdate > DISPLAY_UPDATE_INTERVAL) {
    updateDisplay();
    lastDisplayUpdate = millis();
  }
  
  // Main state machine
  switch (systemState) {
    case 0: stateIdle(); break;
    case 1: stateActive(); break;
    case 2: stateVerify(); break;
    case 3: stateOpen(); break;
    case 4: stateAlert(); break;
  }
  
  // Auto-relock after unlock duration
  if (systemState == 3 && millis() - unlockTime > UNLOCK_DURATION) {
    lockDrawer();
    systemState = 0;
    lcd.clear();
    updateDisplay();
    logAccessAttempt("Auto-relocked");
    delay(1000);
  }
}

void welcomeSequence() {
  // Welcome beep pattern
  beep(200);
  delay(100);
  beep(200);
  delay(100);
  beep(400);
  
  // Welcome display
  lcd.clear();
  lcd.print(" Safe-Hub v2.0 ");
  lcd.setCursor(0, 1);
  lcd.print("  Security ON  ");
  delay(2000);
  
  lcd.clear();
  updateDisplay();
}

void updateDisplay() {
  if (systemState == 0) {
    // Idle state - Show time and status
    lcd.setCursor(0, 0);
    lcd.print("Time: ");
    
    if (rtc.begin()) {
      DateTime now = rtc.now();
      lcd.print(now.hour() < 10 ? "0" : "");
      lcd.print(now.hour());
      lcd.print(":");
      lcd.print(now.minute() < 10 ? "0" : "");
      lcd.print(now.minute());
      lcd.print(":");
      lcd.print(now.second() < 10 ? "0" : "");
      lcd.print(now.second());
    } else {
      lcd.print("--:--:--");
    }
    
    lcd.setCursor(0, 1);
    lcd.print("Status: Locked  ");
    
  } else if (systemState == 1) {
    // Active state - Show code entry
    lcd.setCursor(0, 0);
    lcd.print("Enter Code:     ");
    lcd.setCursor(0, 1);
    lcd.print(">");
    
    // Show entered code as asterisks
    for (int i = 0; i < enteredCode.length(); i++) {
      lcd.print('*');
    }
    
    // Show remaining spaces
    for (int i = enteredCode.length(); i < 6; i++) {
      lcd.print('_');
    }
    
    lcd.print(" ");
    lcd.print(enteredCode.length());
    lcd.print("/6");
    
  } else if (systemState == 3) {
    // Open state - Show countdown
    lcd.setCursor(0, 0);
    lcd.print("*** UNLOCKED ***");
    lcd.setCursor(0, 1);
    unsigned long remaining = (UNLOCK_DURATION - (millis() - unlockTime)) / 1000;
    lcd.print("Closes in: ");
    lcd.print(remaining);
    lcd.print("s  ");
  }
}

void stateIdle() {
  // Check PIR sensor for motion
  if (digitalRead(PIR_PIN) == HIGH && (millis() - lastPIRDetection > PIR_COOLDOWN)) {
    lastPIRDetection = millis();
    
    // Motion detected - activate system
    beep(500);
    delay(50);
    beep(500);
    
    systemState = 1;
    enteredCode = "";
    stateActiveStart = millis();
    
    lcd.clear();
    lcd.print("Motion Detected!");
    lcd.setCursor(0, 1);
    lcd.print("Enter Code...   ");
    
    logAccessAttempt("Motion detected - Code entry");
    delay(1000);
    
    lcd.clear();
    updateDisplay();
  }
}

void stateActive() {
  // Check for timeout (30 seconds)
  if (millis() - stateActiveStart > 30000) {
    systemState = 0;
    lcd.clear();
    lcd.print("Timeout!");
    lcd.setCursor(0, 1);
    lcd.print("System locked  ");
    beep(300);
    delay(100);
    beep(300);
    logAccessAttempt("Timeout - No code entered");
    delay(2000);
    lcd.clear();
    updateDisplay();
    return;
  }
  
  // Read joystick values
  int xValue = analogRead(JOY_X_PIN);
  int yValue = analogRead(JOY_Y_PIN);
  int buttonState = digitalRead(JOY_SW_PIN);
  
  // Detect joystick movements
  static unsigned long lastJoystickTime = 0;
  if (millis() - lastJoystickTime > 300) {
    if (yValue < JOY_THRESHOLD_LOW) {
      addToCode('U'); // Up
      lastJoystickTime = millis();
    } else if (yValue > JOY_THRESHOLD_HIGH) {
      addToCode('D'); // Down
      lastJoystickTime = millis();
    } else if (xValue < JOY_THRESHOLD_LOW) {
      addToCode('L'); // Left
      lastJoystickTime = millis();
    } else if (xValue > JOY_THRESHOLD_HIGH) {
      addToCode('R'); // Right
      lastJoystickTime = millis();
    }
  }
  
  // Check for enter button
  static bool lastButtonState = HIGH;
  static unsigned long lastButtonTime = 0;
  
  if (buttonState == LOW && lastButtonState == HIGH && millis() - lastButtonTime > 500) {
    delay(50);
    if (digitalRead(JOY_SW_PIN) == LOW) {
      beep(150);
      systemState = 2;
      lastButtonTime = millis();
    }
  }
  lastButtonState = buttonState;
}

void stateVerify() {
  lcd.clear();
  lcd.print("Verifying...    ");
  delay(500);
  
  Serial.print("Code Verification: ");
  Serial.print("Entered='");
  Serial.print(enteredCode);
  Serial.print("' Expected='");
  Serial.print(correctCode);
  Serial.println("'");
  
  if (enteredCode == correctCode) {
    // CORRECT CODE - Access Granted
    failedAttempts = 0; // Reset failed attempts
    
    lcd.clear();
    lcd.print(" Access Granted! ");
    lcd.setCursor(0, 1);
    lcd.print("  Welcome User!  ");
    
    // Success beep pattern
    beep(200);
    delay(100);
    beep(300);
    delay(100);
    beep(400);
    
    unlockDrawer();
    systemState = 3;
    unlockTime = millis();
    logAccessAttempt("SUCCESS - Access granted");
    
  } else {
    // WRONG CODE - Access Denied
    failedAttempts++;
    
    lcd.clear();
    lcd.print(" Access Denied!  ");
    lcd.setCursor(0, 1);
    lcd.print(" Wrong Code!     ");
    
    // Error beep pattern
    beep(400);
    delay(200);
    beep(400);
    delay(200);
    beep(400);
    
    logAccessAttempt("FAILED - Wrong code: " + enteredCode + " Attempt: " + String(failedAttempts));
    
    // Check if maximum attempts reached
    if (failedAttempts >= MAX_FAILED_ATTEMPTS) {
      lcd.clear();
      lcd.print("  LOCKED OUT!   ");
      lcd.setCursor(0, 1);
      lcd.print("Too many fails ");
      
      // Alarm sequence
      for (int i = 0; i < 5; i++) {
        beep(500);
        delay(200);
      }
      
      logAccessAttempt("ALARM - Maximum failed attempts reached");
      delay(5000);
      failedAttempts = 0; // Reset after lockout
    }
    
    delay(2000);
    
    // Return to code entry or idle
    if (failedAttempts < MAX_FAILED_ATTEMPTS) {
      systemState = 1;
      enteredCode = "";
      stateActiveStart = millis();
      lcd.clear();
      updateDisplay();
    } else {
      systemState = 0;
      lcd.clear();
      updateDisplay();
    }
  }
}

void stateOpen() {
  // Handled in main loop and updateDisplay()
}

void stateAlert() {
  // Handled in stateVerify
}

void addToCode(char direction) {
  if (enteredCode.length() < 6) {
    enteredCode += direction;
    beep(80); // Keypress feedback
    updateDisplay();
    Serial.print("Code entry: ");
    Serial.println(enteredCode);
  } else {
    // Code full - error beep
    beep(50);
    delay(50);
    beep(50);
  }
}

void beep(int duration) {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(duration);
  digitalWrite(BUZZER_PIN, LOW);
}

void unlockDrawer() {
  lockServo.write(90);  // Unlock position
  Serial.println("Drawer UNLOCKED");
}

void lockDrawer() {
  lockServo.write(0);   // Lock position
  Serial.println("Drawer LOCKED");
}

void logAccessAttempt(String event) {
  Serial.println("=== SECURITY LOG ===");
  
  // Timestamp
  if (rtc.begin()) {
    DateTime now = rtc.now();
    Serial.print("Time: ");
    Serial.print(now.hour() < 10 ? "0" : "");
    Serial.print(now.hour());
    Serial.print(":");
    Serial.print(now.minute() < 10 ? "0" : "");
    Serial.print(now.minute());
    Serial.print(":");
    Serial.print(now.second() < 10 ? "0" : "");
    Serial.println(now.second());
  }
  
  Serial.print("Event: ");
  Serial.println(event);
  Serial.println("===================");
}
