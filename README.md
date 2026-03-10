# Keyless-Electronic-Door-Lock
This is a group project that I worked on in an Intro to Engineering class in Fall 2025. The project consists of an Arduino Mega board, breadboard, RFID reader, number keypad, LCD display, servo motor, and a door lock. The project allows for a door lock to be opened without a key by using a password on the keypad or a rfid card with a recongnized id.

In addition to this I have a demonstration video of the project in action: https://www.youtube.com/watch?v=QHXwPzO9o_8

On this page I have screenshots of images of the project in different stages of development. I would like to share the code used for the project as well. The code was written by me in the Arduino IDE.

#include <MemoryDumper.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <MFRC522.h>
#include <Servo.h>
#include <Keypad.h>

// Initialize LCD with I2C address (usually 0x27 or 0x3F)
LiquidCrystal_I2C lcd(0x27, 16, 2);

// RFID setup
#define SS_PIN 53
#define RST_PIN 5
MFRC522 rfid(SS_PIN, RST_PIN);

// Servo setup
Servo doorServo;

// Keypad setup
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1', '2', '3', 'A'},
  {'4', '5', '6', 'B'},
  {'7', '8', '9', 'C'},
  {'*', '0', '#', 'D'}
};
byte rowPins[ROWS] = {33, 35, 37, 39};
byte colPins[COLS] = {31, 29, 27, 25};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// --------------------------------------------
// MULTIPLE RFID CARDS HERE 👇
String authorizedUIDs[] = {
  "898FA011",  // Card 1
  "C11C7FA2"   // Card 2
};
const byte totalCards = sizeof(authorizedUIDs) / sizeof(authorizedUIDs[0]);
// --------------------------------------------

String correctPassword = "456";  // Keypad password
String inputPassword = "";
bool doorOpen = false;

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();

  doorServo.attach(9);
  doorServo.write(0);  // Locked position

  lcd.begin(16, 2);
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Door Lock System");

  delay(2000);
  lcd.clear();
  lcd.print("Scan Card or");
  lcd.setCursor(0, 1);
  lcd.print("Enter Password");
}

void loop() {
  // --- RFID Card Check ---
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    String uid = "";
    for (byte i = 0; i < rfid.uid.size; i++) {
      String hexByte = String(rfid.uid.uidByte[i], HEX);
      if (hexByte.length() == 1) hexByte = "0" + hexByte;
      uid += hexByte;
    }
    uid.toUpperCase();

    Serial.print("Card UID: ");
    Serial.println(uid);

    if (isAuthorizedCard(uid)) {
      openDoor();
    } else {
      lcd.clear();
      lcd.print("Access Denied!");
      delay(2000);
      showIdleScreen();
    }
    rfid.PICC_HaltA();
  }

  // --- Keypad Input Check ---
  char key = keypad.getKey();
  if (key) {
    lcd.setCursor(0, 1);
    if (key == '#') {
      if (inputPassword == correctPassword) {
        openDoor();
      } else {
        lcd.clear();
        lcd.print("Incorrect Pass");
        delay(2000);
        lcd.clear();
        lcd.print("Enter Password");
      }
      inputPassword = "";  // Reset after attempt
    } else if (key == '*') {
      inputPassword = "";
      lcd.clear();
      lcd.print("Enter Password");
    } else {
      inputPassword += key;
      lcd.print("*");  // Mask input
    }
  }
}

// --------------------------------------------------
// Helper: Check if scanned UID is authorized
// --------------------------------------------------
bool isAuthorizedCard(String uid) {
  for (byte i = 0; i < totalCards; i++) {
    if (uid == authorizedUIDs[i]) {
      return true;
    }
  }
  return false;
}

// --------------------------------------------------
// Door control functions
// --------------------------------------------------
void openDoor() {
  lcd.clear();
  lcd.print("Access Granted!");
  doorServo.write(90);  // Unlock angle
  delay(2000);
  doorServo.write(0);   // Lock again
  lcd.clear();
  lcd.print("Door Locked");
  delay(1000);
  showIdleScreen();
}

void showIdleScreen() {
  lcd.clear();
  lcd.print("Scan Card or");
  lcd.setCursor(0, 1);
  lcd.print("Enter Password");
}
