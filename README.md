#include <IRremote.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <SoftwareSerial.h>
#include <Stepper.h>

const int STEPS_PER_REV = 2048;
const int LOCK_STEPS = 150;

Stepper lockMotor(STEPS_PER_REV, A0, A2, A1, A3);

const int IR_PIN = 3;
const int CEILING_PIN = 7;
const int BOOKSHELF_PIN = 4;
const int RGB_PIN = 6;
const int FAN1_PIN = 5;
const int BUZZER_PIN = 8;
const int FAN2_PIN = 9;

SoftwareSerial espSerial(10, 11);

const int BUTTON_1 = 0x45;
const int BUTTON_2 = 0x46;
const int BUTTON_3 = 0x47;
const int BUTTON_4 = 0x44;
const int BUTTON_5 = 0x40;
const int BUTTON_6 = 0x43;
const int BUTTON_7 = 0x07;
const int BUTTON_STAR = 0x16;
const int BUTTON_HASH_A = 0x52;
const int BUTTON_HASH_B = 0x15;
const int BUTTON_HASH_C = 0x0D;
const int BUTTON_RIGHT = 0x5A;
const int BUTTON_LEFT = 0x08;

int activeMode = 1;

bool fan1State = false;
bool bookshelfState = false;
bool ceilingState = false;
bool rgbState = false;
bool fan2State = false;
bool isLocked = false;

String receivedData = "";
String espReceivedData = "";

String pcTime = "00:00";
int cpuTemp = 0, cpuLoad = 0;
int gpuTemp = 0, gpuLoad = 0;

int highTempCount = 0;
const int TEMP_THRESHOLD = 5;

unsigned long screenUpdateTime = 0;
unsigned long animationTime = 0;
int animationStep = 0;

String lastBeepMinute = "";
unsigned long nextGameBeep = 3600000;
unsigned long gameStartTime = 0;

char fanAnimation[4] = {'|', '/', '-', '\\'};

unsigned long messageTime = 0;
bool messageActive = false;

LiquidCrystal_I2C mainLCD(0x27, 16, 2);
LiquidCrystal_I2C infoLCD(0x26, 16, 2);

void turnAllOff();
void releaseMotor();
void buzzerSignal(int count);
void refreshModeScreen();
void updateStatusLine();
void setupStandardMode();
void setupGameMode();
void setupWakeMode();
void sleepMode();
void showMessage(String device, bool state);
void processESPCommand(String command);
void processPythonData(String data);

void setup() {
  Serial.begin(9600);
  espSerial.begin(9600);

  lockMotor.setSpeed(14);

  pinMode(FAN1_PIN, OUTPUT);
  pinMode(BOOKSHELF_PIN, OUTPUT);
  pinMode(CEILING_PIN, OUTPUT);
  pinMode(RGB_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(FAN2_PIN, OUTPUT);

  turnAllOff();
  releaseMotor();

  IrReceiver.begin(IR_PIN, ENABLE_LED_FEEDBACK);

  mainLCD.init();
  mainLCD.backlight();
  mainLCD.print("SYSTEM READY");

  infoLCD.init();
  infoLCD.backlight();
  infoLCD.print("INFO PANEL");

  buzzerSignal(2);
  delay(1000);

  setupStandardMode();
}

void loop() {
  unsigned long currentTime = millis();

  while (espSerial.available() > 0) {
    char c = espSerial.read();

    if (c == '\n') {
      espReceivedData.trim();

      if (espReceivedData.length() > 0) {
        processESPCommand(espReceivedData);
      }

      espReceivedData = "";
    } else if (c != '\r') {
      espReceivedData += c;
    }
  }

  if (activeMode == 0) {
    if (IrReceiver.decode()) {
      uint16_t command = IrReceiver.decodedIRData.command;

      if (command == BUTTON_1) {
        activeMode = 1;
        setupStandardMode();
      } else if (command == BUTTON_2) {
        activeMode = 2;
        gameStartTime = millis();
        nextGameBeep = 3600000;
        setupGameMode();
      } else if (command == BUTTON_HASH_A ||
                 command == BUTTON_HASH_B ||
                 command == BUTTON_HASH_C ||
                 command == BUTTON_STAR) {
        activeMode = 3;
        setupWakeMode();
      } else if (command == BUTTON_RIGHT && isLocked) {
        lockMotor.step(-LOCK_STEPS);
        releaseMotor();
        isLocked = false;
        buzzerSignal(1);
      } else if (command == BUTTON_LEFT && !isLocked) {
        lockMotor.step(LOCK_STEPS);
        releaseMotor();
        isLocked = true;
        buzzerSignal(1);
      }

      IrReceiver.resume();
    }

    return;
  }

  if (messageActive && currentTime - messageTime >= 2000) {
    messageActive = false;
    refreshModeScreen();
  }

  if (!messageActive && activeMode == 2 &&
      currentTime - animationTime >= 300) {
    animationTime = currentTime;

    if (fan1State || fan2State) {
      animationStep++;

      if (animationStep > 3) {
        animationStep = 0;
      }
    } else {
      animationStep = 0;
    }

    updateStatusLine();
  }

  if (Serial.available() > 0 && activeMode != 3) {
    char c = Serial.read();

    if (c == '\n') {
      processPythonData(receivedData);
      receivedData = "";
    } else if (c != '\r') {
      receivedData += c;
    }
  }

  if (activeMode == 2) {
    unsigned long elapsedTime = currentTime - gameStartTime;

    if (elapsedTime >= nextGameBeep) {
      buzzerSignal(2);
      nextGameBeep += 3600000;
    }
  }

  if (currentTime - screenUpdateTime >= 1000 && activeMode != 3) {
    screenUpdateTime = currentTime;

    infoLCD.setCursor(0, 0);
    infoLCD.print("Time: " + pcTime + "      ");

    infoLCD.setCursor(0, 1);
    infoLCD.print("CPU: ");
    infoLCD.print(cpuTemp);
    infoLCD.print((char)223);
    infoLCD.print("C %");
    infoLCD.print(cpuLoad);
    infoLCD.print("    ");
  }

  if (IrReceiver.decode()) {
    uint16_t command = IrReceiver.decodedIRData.command;

    if (command == BUTTON_1) {
      activeMode = 1;
      setupStandardMode();
    } else if (command == BUTTON_2) {
      activeMode = 2;
      gameStartTime = millis();
      nextGameBeep = 3600000;
      setupGameMode();
    } else if (command == BUTTON_HASH_A ||
               command == BUTTON_HASH_B ||
               command == BUTTON_HASH_C) {
      activeMode = 3;
      setupWakeMode();
    } else if (command == BUTTON_STAR) {
      sleepMode();
    } else if (command == BUTTON_RIGHT) {
      if (isLocked) {
        showMessage("Door Lock", false);
        lockMotor.step(-LOCK_STEPS);
        releaseMotor();
        isLocked = false;
      } else {
        buzzerSignal(3);
      }
    } else if (command == BUTTON_LEFT) {
      if (!isLocked) {
        showMessage("Door Lock", true);
        lockMotor.step(LOCK_STEPS);
        releaseMotor();
        isLocked = true;
      } else {
        buzzerSignal(3);
      }
    } else if (command == BUTTON_3) {
      if (activeMode == 3) activeMode = 1;

      bookshelfState = !bookshelfState;
      digitalWrite(BOOKSHELF_PIN, bookshelfState ? LOW : HIGH);
      showMessage("Bookshelf", bookshelfState);
    } else if (command == BUTTON_4) {
      if (activeMode == 3) activeMode = 1;

      ceilingState = !ceilingState;
      digitalWrite(CEILING_PIN, ceilingState ? LOW : HIGH);
      showMessage("Ceiling", ceilingState);
    } else if (command == BUTTON_5) {
      if (activeMode == 3) activeMode = 1;

      if (fan2State) {
        buzzerSignal(3);
      } else {
        fan1State = !fan1State;
        digitalWrite(FAN1_PIN, fan1State ? LOW : HIGH);
        showMessage("Fan 1", fan1State);
      }
    } else if (command == BUTTON_6) {
      if (activeMode == 3) activeMode = 1;

      fan2State = !fan2State;

      if (fan2State) {
        fan1State = false;
        digitalWrite(FAN1_PIN, HIGH);
        delay(50);
      }

      digitalWrite(FAN2_PIN, fan2State ? LOW : HIGH);
      showMessage("Fan 2", fan2State);
    } else if (command == BUTTON_7) {
      if (activeMode == 3) activeMode = 1;

      rgbState = !rgbState;
      digitalWrite(RGB_PIN, rgbState ? LOW : HIGH);
      showMessage("RGB Light", rgbState);
    }

    IrReceiver.resume();
  }
}

void processPythonData(String data) {
  espSerial.println(data);

  int comma1 = data.indexOf(',');
  int comma2 = data.indexOf(',', comma1 + 1);
  int comma3 = data.indexOf(',', comma2 + 1);
  int comma4 = data.indexOf(',', comma3 + 1);

  if (comma1 != -1 && comma2 != -1 &&
      comma3 != -1 && comma4 != -1) {
    pcTime = data.substring(0, comma1);
    cpuTemp = data.substring(comma1 + 1, comma2).toInt();
    cpuLoad = data.substring(comma2 + 1, comma3).toInt();
    gpuTemp = data.substring(comma3 + 1, comma4).toInt();
    gpuLoad = data.substring(comma4 + 1).toInt();

    if (activeMode == 2) {
      if (cpuTemp >= 60) {
        highTempCount++;

        if (highTempCount >= TEMP_THRESHOLD && !fan2State) {
          fan2State = true;
          fan1State = false;

          digitalWrite(FAN1_PIN, HIGH);
          delay(10);
          digitalWrite(FAN2_PIN, LOW);

          showMessage("Auto Fan 2", true);
        }
      } else if (cpuTemp < 50) {
        if (fan2State) {
          fan2State = false;
          fan1State = true;

          digitalWrite(FAN2_PIN, HIGH);
          delay(10);
          digitalWrite(FAN1_PIN, LOW);

          showMessage("Auto Fan 1", true);
        }

        highTempCount = 0;
      } else {
        highTempCount = 0;
      }
    } else {
      highTempCount = 0;
    }

    if (pcTime.length() >= 5) {
      String minute = pcTime.substring(3, 5);

      if (minute == "00" && lastBeepMinute != "00" &&
          activeMode != 2) {
        buzzerSignal(2);
        lastBeepMinute = "00";
      } else if (minute != "00") {
        lastBeepMinute = minute;
      }
    }
  }
}

void processESPCommand(String command) {
  if (command == "ESP_MOD_STD") {
    activeMode = 1;
    setupStandardMode();
  } else if (command == "ESP_MOD_GAME") {
    activeMode = 2;
    gameStartTime = millis();
    nextGameBeep = 3600000;
    setupGameMode();
  } else if (command == "ESP_UYANDIR" || command == "ESP_MOD_UYAN") {
    activeMode = 3;
    setupWakeMode();
  } else if (command == "ESP_MOD_STD_KAPAT" ||
             command == "ESP_MOD_GAME_KAPAT") {
    buzzerSignal(1);
    turnAllOff();
    refreshModeScreen();
  } else if (command == "ESP_KILIT_AC") {
    if (isLocked) {
      showMessage("Door Lock", false);
      lockMotor.step(-LOCK_STEPS);
      releaseMotor();
      isLocked = false;
    } else {
      buzzerSignal(3);
    }
  } else if (command == "ESP_KILIT_VUR") {
    if (!isLocked) {
      showMessage("Door Lock", true);
      lockMotor.step(LOCK_STEPS);
      releaseMotor();
      isLocked = true;
    } else {
      buzzerSignal(3);
    }
  } else {
    if (activeMode == 3 && command != "ESP_UYKU") {
      activeMode = 1;
      setupStandardMode();
    }

    if (command == "ESP_TAVAN") {
      ceilingState = !ceilingState;
      digitalWrite(CEILING_PIN, ceilingState ? LOW : HIGH);
      showMessage("Ceiling", ceilingState);
    } else if (command == "ESP_KITAPLIK") {
      bookshelfState = !bookshelfState;
      digitalWrite(BOOKSHELF_PIN, bookshelfState ? LOW : HIGH);
      showMessage("Bookshelf", bookshelfState);
    } else if (command == "ESP_FAN1") {
      if (fan2State) {
        buzzerSignal(3);
      } else {
        fan1State = !fan1State;
        digitalWrite(FAN1_PIN, fan1State ? LOW : HIGH);
        showMessage("Fan 1", fan1State);
      }
    } else if (command == "ESP_FAN2") {
      fan2State = !fan2State;

      if (fan2State) {
        fan1State = false;
        digitalWrite(FAN1_PIN, HIGH);
        delay(10);
      }

      digitalWrite(FAN2_PIN, fan2State ? LOW : HIGH);
      showMessage("Fan 2", fan2State);
    } else if (command == "ESP_RGB") {
      rgbState = !rgbState;
      digitalWrite(RGB_PIN, rgbState ? LOW : HIGH);
      showMessage("RGB Light", rgbState);
    } else if (command == "ESP_UYKU") {
      sleepMode();
    }
  }
}

void releaseMotor() {
  digitalWrite(A0, LOW);
  digitalWrite(A1, LOW);
  digitalWrite(A2, LOW);
  digitalWrite(A3, LOW);
}

void refreshModeScreen() {
  mainLCD.clear();
  infoLCD.clear();

  mainLCD.backlight();
  infoLCD.backlight();

  if (activeMode == 1) {
    mainLCD.setCursor(0, 0);
    mainLCD.print("STANDARD MODE");
    updateStatusLine();
  } else if (activeMode == 2) {
    mainLCD.setCursor(0, 0);
    mainLCD.print("GAME MODE");
    updateStatusLine();
  } else if (activeMode == 3) {
    mainLCD.setCursor(0, 0);
    mainLCD.print("SYSTEM READY");
    mainLCD.setCursor(0, 1);
    mainLCD.print("STANDBY...");
    infoLCD.setCursor(0, 0);
    infoLCD.print("PANEL ACTIVE");
  }
}

void updateStatusLine() {
  mainLCD.setCursor(0, 1);

  if (activeMode == 2) {
    mainLCD.print("Fan: ");

    if (fan2State) {
      for (int i = 0; i < 4; i++) {
        mainLCD.print(fanAnimation[animationStep]);
      }
      mainLCD.print("          ");
    } else if (fan1State) {
      for (int i = 0; i < 2; i++) {
        mainLCD.print(fanAnimation[animationStep]);
      }
      mainLCD.print("            ");
    } else {
      mainLCD.print("| OFF          ");
    }
  } else {
    int lightsOn = 0;

    if (bookshelfState) lightsOn++;
    if (ceilingState) lightsOn++;

    mainLCD.print("Lights 2/");
    mainLCD.print(lightsOn);
    mainLCD.print("   ");
  }
}

void showMessage(String device, bool state) {
  buzzerSignal(1);

  mainLCD.clear();
  mainLCD.setCursor(0, 0);
  mainLCD.print("ACTION COMPLETE:");
  mainLCD.setCursor(0, 1);
  mainLCD.print(device);
  mainLCD.print(state ? ": ON/LOCKED" : ": OFF/UNLOCKED");

  messageTime = millis();
  messageActive = true;
}

void setupStandardMode() {
  mainLCD.backlight();
  infoLCD.backlight();
  buzzerSignal(1);

  bookshelfState = true;
  ceilingState = true;
  rgbState = false;
  fan1State = false;

  digitalWrite(BOOKSHELF_PIN, LOW);
  delay(50);
  digitalWrite(CEILING_PIN, LOW);
  delay(50);
  digitalWrite(RGB_PIN, HIGH);
  digitalWrite(FAN1_PIN, HIGH);

  refreshModeScreen();
}

void setupGameMode() {
  mainLCD.backlight();
  infoLCD.backlight();
  buzzerSignal(1);

  fan2State = false;
  digitalWrite(FAN2_PIN, HIGH);
  delay(50);

  bookshelfState = false;
  ceilingState = false;
  rgbState = true;
  fan1State = true;

  digitalWrite(BOOKSHELF_PIN, HIGH);
  delay(50);
  digitalWrite(CEILING_PIN, HIGH);
  delay(50);
  digitalWrite(RGB_PIN, LOW);
  delay(50);
  digitalWrite(FAN1_PIN, LOW);

  refreshModeScreen();
}

void setupWakeMode() {
  buzzerSignal(1);
  turnAllOff();
  refreshModeScreen();
}

void sleepMode() {
  buzzerSignal(2);
  activeMode = 0;
  turnAllOff();

  mainLCD.clear();
  mainLCD.setCursor(0, 0);
  mainLCD.print("SLEEP MODE...");
  mainLCD.setCursor(0, 1);
  mainLCD.print("GOOD NIGHT...");

  infoLCD.clear();
  infoLCD.print("SYSTEM OFF...");

  delay(2000);

  mainLCD.noBacklight();
  infoLCD.noBacklight();
}

void turnAllOff() {
  digitalWrite(FAN1_PIN, HIGH);
  delay(20);
  digitalWrite(BOOKSHELF_PIN, HIGH);
  delay(20);
  digitalWrite(CEILING_PIN, HIGH);
  delay(20);
  digitalWrite(RGB_PIN, HIGH);
  delay(20);
  digitalWrite(FAN2_PIN, HIGH);

  bookshelfState = false;
  ceilingState = false;
  rgbState = false;
  fan1State = false;
  fan2State = false;
}

void buzzerSignal(int count) {
  for (int i = 0; i < count; i++) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(80);
    digitalWrite(BUZZER_PIN, LOW);

    if (count > 1) {
      delay(80);
    }
  }
}
