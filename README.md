# my-arduino-project
ini code untuk project c.a.p gua di arduino atau MCU lain. (11/9/2026)

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SH110X.h>
#include <FluxGarage_RoboEyes.h>

// ======================================
// OLED
// ======================================

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDRESS 0x3C

Adafruit_SH1106G display(
  SCREEN_WIDTH,
  SCREEN_HEIGHT,
  &Wire,
  -1
);

// ======================================
// ROBO EYES
// ======================================

RoboEyes<Adafruit_SH1106G> roboEyes(display);

// ======================================
// HC-SR04
// ======================================

#define TRIG_PIN 9
#define ECHO_PIN 8

float distanceCm = -1;
float rawDistance = -1;

unsigned long sensorTimer = 0;

const unsigned long SENSOR_INTERVAL = 100;

int invalidReadings = 0;

bool objectDetectedState = false;

// ======================================
// DASHBOARD BUTTON
// ======================================

#define BUTTON_PIN 4

bool infoScreen = false;

int buttonState = HIGH;
int lastButtonReading = HIGH;

unsigned long debounceTimer = 0;
unsigned long dashboardTimer = 0;

const unsigned long DEBOUNCE_TIME = 40;
const unsigned long DASHBOARD_INTERVAL = 250;

// ======================================
// POWER BUTTON
// ======================================

#define POWER_BUTTON_PIN 5

enum PowerState {
  POWER_START_MESSAGE,
  POWER_RUNNING,
  POWER_SHUTDOWN_CLOSING,
  POWER_SHUTDOWN_BLANK,
  POWER_SHUTDOWN_MESSAGE,
  POWER_OFF,
  POWER_WAKE_MESSAGE
};

PowerState powerState = POWER_START_MESSAGE;

int powerButtonState = HIGH;
int lastPowerReading = HIGH;

unsigned long powerDebounceTimer = 0;
unsigned long powerTimer = 0;

const unsigned long EYE_CLOSE_DURATION = 600;
const unsigned long POWER_DEBOUNCE = 40;
const unsigned long MESSAGE_DURATION = 1200;
const unsigned long BLANK_DURATION = 350;
const unsigned long FAREWELL_DURATION = 1300;

// ======================================
// BEHAVIOR SYSTEM
// ======================================

enum RobotMode {
  NORMAL_MODE,
  CURIOUS_MODE,
  TIRED_MODE
};

RobotMode currentMode = NORMAL_MODE;

// Deklarasi awal fungsi
void changeMode(RobotMode newMode);

unsigned long interactionTimer = 0;
unsigned long idleTimer = 0;
unsigned long eyeMovementTimer = 0;

bool interacting = false;

const unsigned long CURIOUS_TIME = 5000;
const unsigned long TIRED_TIME = 5000;

const unsigned long EYE_INTERVAL = 700;

// ======================================
// MOOD CONTROLLER
// ======================================

int lastMood = -1;

void changeMood(int mood) {

  if (mood != lastMood) {

    roboEyes.setMood(mood);
    lastMood = mood;

  }
}

// ======================================
// POWER DISPLAY
// ======================================

void showGreeting() {

  display.clearDisplay();

  display.setTextSize(2);
  display.setTextColor(SH110X_WHITE);

  display.setCursor(10, 24);
  display.println(F("HI LOSER!"));

  display.display();

}

void showFarewell() {

  display.clearDisplay();

  display.setTextSize(1);
  display.setTextColor(SH110X_WHITE);

  display.setCursor(19, 28);
  display.println(F("MET TINGGAL CUY"));

  display.display();

}

// ======================================
// POWER BUTTON CONTROLLER
// ======================================

void handlePowerButton() {

  int reading = digitalRead(POWER_BUTTON_PIN);

  if (reading != lastPowerReading) {

    powerDebounceTimer = millis();

  }

  if (millis() - powerDebounceTimer > POWER_DEBOUNCE) {

    if (reading != powerButtonState) {

      powerButtonState = reading;

      if (powerButtonState == LOW) {

        if (powerState == POWER_RUNNING) {

  // Sembunyikan dashboard jika sedang aktif
  infoScreen = false;

  // Matikan autoblinker agar tidak mengganggu
  roboEyes.setAutoblinker(OFF, 3, 2);

  // Tutup kedua mata
  roboEyes.close();

  powerState = POWER_SHUTDOWN_CLOSING;
  powerTimer = millis();

}

        // WAKE UP
        else if (powerState == POWER_OFF) {

          // Reset sensor
          distanceCm = -1;
          rawDistance = -1;

          invalidReadings = 0;
          objectDetectedState = false;

          sensorTimer = millis();

          // Reset interaction
          interacting = false;
          interactionTimer = 0;
          idleTimer = millis();

          // Reset dashboard
          infoScreen = false;
          dashboardTimer = 0;

          // Reset robot mode
          currentMode = TIRED_MODE;
          lastMood = -1;

          changeMode (NORMAL_MODE);

          // Show greeting
          showGreeting();

          powerState = POWER_WAKE_MESSAGE;
          powerTimer = millis();

        }

      }

    }

  }

  lastPowerReading = reading;

}

// ======================================
// POWER SEQUENCE
// ======================================

void updatePowerSequence() {

  unsigned long now = millis();

  // Initial greeting
  if (powerState == POWER_START_MESSAGE) {

    if (now - powerTimer >= MESSAGE_DURATION) {

      roboEyes.open();

      powerState = POWER_RUNNING;
      idleTimer = now;

    }

  }

  // ==================================
  // EYES CLOSING
  // ==================================

  else if (powerState == POWER_SHUTDOWN_CLOSING) {

    // Terus jalankan animasi penutupan mata
    roboEyes.update();

    if (now - powerTimer >= EYE_CLOSE_DURATION) {

      display.clearDisplay();
      display.display();

      powerState = POWER_SHUTDOWN_BLANK;
      powerTimer = now;

    }

  }

  // Blank before farewell
  else if (powerState == POWER_SHUTDOWN_BLANK) {

    if (now - powerTimer >= BLANK_DURATION) {

      showFarewell();

      powerState = POWER_SHUTDOWN_MESSAGE;
      powerTimer = now;

    }

  }

  // Farewell message
  else if (powerState == POWER_SHUTDOWN_MESSAGE) {

    if (now - powerTimer >= FAREWELL_DURATION) {

      display.clearDisplay();
      display.display();

      powerState = POWER_OFF;

    }

  }

  // Wake greeting
  else if (powerState == POWER_WAKE_MESSAGE) {

    if (now - powerTimer >= MESSAGE_DURATION) {

      roboEyes.setAutoblinker(ON, 3, 2);
      roboEyes.open();

      powerState = POWER_RUNNING;
      idleTimer = now;

    }

  }

}

  

  

// ======================================
// SENSOR RELIABILITY
// ======================================

void readSensor() {

  if (millis() - sensorTimer < SENSOR_INTERVAL) {
    return;
  }

  sensorTimer = millis();

  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);

  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);

  digitalWrite(TRIG_PIN, LOW);

  unsigned long duration = pulseIn(
    ECHO_PIN,
    HIGH,
    9000UL
  );

  if (duration == 0) {

    invalidReadings++;

    if (invalidReadings >= 3) {
      distanceCm = -1;
    }

    return;
  }

  invalidReadings = 0;

  rawDistance = duration * 0.0343 / 2;

  // Exponential smoothing
  if (distanceCm < 0) {

    distanceCm = rawDistance;

  } else {

    distanceCm =
      (distanceCm * 0.65) +
      (rawDistance * 0.35);

  }

  // Hysteresis
  if (!objectDetectedState) {

    if (distanceCm > 0 && distanceCm <= 19) {
      objectDetectedState = true;
    }

  } else {

    if (distanceCm < 0 || distanceCm >= 22) {
      objectDetectedState = false;
    }

  }

}

// ======================================
// MODE CONTROLLER
// ======================================

void changeMode(RobotMode newMode) {

  if (newMode == currentMode) {
    return;
  }

  currentMode = newMode;

  if (newMode == NORMAL_MODE) {

    roboEyes.setSweat(OFF);
    roboEyes.setCuriosity(OFF);

    roboEyes.setPosition(DEFAULT);

    changeMood(DEFAULT);

  }

  else if (newMode == CURIOUS_MODE) {

    roboEyes.setSweat(OFF);
    roboEyes.setCuriosity(ON);

    changeMood(HAPPY);

    eyeMovementTimer = millis();

  }

  else if (newMode == TIRED_MODE) {

    roboEyes.setCuriosity(OFF);
    roboEyes.setSweat(ON);

    changeMood(TIRED);

  }

}

// ======================================
// CURIOUS EYE MOVEMENT
// ======================================

void updateCuriousEyes() {

  if (currentMode != CURIOUS_MODE) {
    return;
  }

  if (millis() - eyeMovementTimer < EYE_INTERVAL) {
    return;
  }

  eyeMovementTimer = millis();

  int directions[] = {
    N, NE, E, SE, S, SW, W, NW
  };

  int randomDirection = random(0, 8);

  roboEyes.setPosition(
    directions[randomDirection]
  );

}

// ======================================
// EMOTION INTELLIGENCE
// ======================================

void updateEmotion() {

  unsigned long now = millis();

  bool objectDetected = objectDetectedState;

  // OBJECT DETECTED
  if (objectDetected) {

    idleTimer = 0;

    if (!interacting) {

      interacting = true;
      interactionTimer = now;

      changeMode(NORMAL_MODE);

      return;
    }

    // Interaction exceeds 5 seconds
    if (now - interactionTimer >= TIRED_TIME) {

      changeMode(TIRED_MODE);
      return;

    }

    changeMode(NORMAL_MODE);

    if (distanceCm < 5) {

      changeMood(ANGRY);

    } else {

      changeMood(DEFAULT);

    }

  }

  // NO OBJECT DETECTED
  else {

    if (interacting) {

      interacting = false;
      idleTimer = now;

    }

    if (idleTimer == 0) {
      idleTimer = now;
    }

    // Curious after 5 seconds
    if (now - idleTimer >= CURIOUS_TIME) {

      changeMode(CURIOUS_MODE);

    } else {

      changeMode(NORMAL_MODE);
      changeMood(HAPPY);

    }

  }

}

// ======================================
// DASHBOARD BUTTON
// ======================================

void handleButton() {

  int reading = digitalRead(BUTTON_PIN);

  if (reading != lastButtonReading) {

    debounceTimer = millis();

  }

  if (millis() - debounceTimer > DEBOUNCE_TIME) {

    if (reading != buttonState) {

      buttonState = reading;

      if (buttonState == LOW) {

        infoScreen = !infoScreen;

        dashboardTimer = 0;

      }

    }

  }

  lastButtonReading = reading;

}

// ======================================
// SYSTEM INFORMATION
// ======================================

void updateDashboard() {

  if (!infoScreen) {
    return;
  }

  if (millis() - dashboardTimer < DASHBOARD_INTERVAL) {
    return;
  }

  dashboardTimer = millis();

  display.clearDisplay();

  display.setTextSize(1);
  display.setTextColor(SH110X_WHITE);

  // Title
  display.setCursor(0, 0);
  display.println(F("A.E.P. SYSTEM INFO"));

  display.drawLine(0, 10, 127, 10, SH110X_WHITE);

  // Uptime
  unsigned long uptime = millis() / 1000;

  unsigned int hours = uptime / 3600;
  unsigned int minutes = (uptime % 3600) / 60;
  unsigned int seconds = uptime % 60;

  display.setCursor(0, 15);
  display.print(F("UPTIME: "));

  if (hours < 10) display.print('0');
  display.print(hours);
  display.print(':');

  if (minutes < 10) display.print('0');
  display.print(minutes);
  display.print(':');

  if (seconds < 10) display.print('0');
  display.println(seconds);

  // Distance
  display.setCursor(0, 27);
  display.print(F("DISTANCE: "));

  if (distanceCm >= 0) {

    display.print(distanceCm, 1);
    display.println(F(" cm"));

  } else {

    display.println(F("NO ECHO"));

  }

  // Sensor status
  display.setCursor(0, 39);
  display.print(F("SENSOR: "));

  if (invalidReadings >= 3) {
    display.println(F("UNSTABLE"));
  } else {
    display.println(F("ACTIVE"));
  }

  // Robot mode
  display.setCursor(0, 51);
  display.print(F("MODE: "));

  if (currentMode == NORMAL_MODE) {
    display.println(F("NORMAL"));
  }

  else if (currentMode == CURIOUS_MODE) {
    display.println(F("CURIOUS"));
  }

  else if (currentMode == TIRED_MODE) {
    display.println(F("TIRED"));
  }

  display.display();

}

// ======================================
// SETUP
// ======================================

void setup() {

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(POWER_BUTTON_PIN, INPUT_PULLUP);

  digitalWrite(TRIG_PIN, LOW);

  display.begin(OLED_ADDRESS, true);

  roboEyes.begin(128, 64, 120);

  roboEyes.setAutoblinker(ON, 3, 2);

  roboEyes.setWidth(38, 38);
  roboEyes.setHeight(38, 38);

  roboEyes.setBorderradius(10, 10);
  roboEyes.setSpacebetween(12);

  roboEyes.setMood(DEFAULT);

  randomSeed(analogRead(A0));

  idleTimer = millis();

  // Initial greeting
  showGreeting();

  powerTimer = millis();

}

// ======================================
// MAIN LOOP
// ======================================

void loop() {

  // Power button always active
  handlePowerButton();

  updatePowerSequence();

  // Stop robot processing during standby/messages
  if (powerState != POWER_RUNNING) {
    return;
  }

  // Main robot system
  readSensor();

  handleButton();

  updateEmotion();

  updateCuriousEyes();

  if (infoScreen) {

    updateDashboard();

  } else {

    roboEyes.update();

  }

}
