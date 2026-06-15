#include <SPI.h>
#include <RF24.h>
#include <EEPROM.h>

#define CE_PIN 9
#define CSN_PIN 10

RF24 radio(CE_PIN, CSN_PIN);
const byte pipeRX[6] = "TXA01";
const byte pipeTX[6] = "RXA01";

/* ---------- PINS ---------- */
uint8_t RELAY_PIN[2]  = {3, 4};
uint8_t SWITCH_PIN[2] = {5, 6};

/* ---------- STATES ---------- */
bool loadState[2];

/* ---------- SWITCH DEBOUNCE ---------- */
bool lastReading[2] = {HIGH, HIGH};
bool stableState[2] = {HIGH, HIGH};
unsigned long lastDebounce[2] = {0, 0};
const unsigned long debounceDelay = 50;

/* ---------- RELAY ---------- */
void applyRelay(uint8_t i) {
  // Active-LOW relay module
  digitalWrite(RELAY_PIN[i], loadState[i] ? LOW : HIGH);
}

/* ---------- EEPROM ---------- */
void loadSaved() {
  loadState[0] = EEPROM.read(0) == 1;
  loadState[1] = EEPROM.read(1) == 1;

  applyRelay(0);
  applyRelay(1);
}

void saveState() {
  EEPROM.update(0, loadState[0]);
  EEPROM.update(1, loadState[1]);
}

/* ---------- RF ---------- */
void sendState() {
  uint8_t d = (loadState[0] ? 1 : 0) | (loadState[1] ? 2 : 0);
  uint8_t pkt = (0x8 << 4) | d;

  radio.stopListening();
  radio.write(&pkt, 1);
  radio.startListening();
}

/* ---------- SETUP ---------- */
void setup() {
  Serial.begin(115200);

  for (int i = 0; i < 2; i++) {
    pinMode(RELAY_PIN[i], OUTPUT);
    pinMode(SWITCH_PIN[i], INPUT_PULLUP);
  }

  loadSaved();

  radio.begin();
  radio.setPALevel(RF24_PA_HIGH);
  radio.setDataRate(RF24_250KBPS);
  radio.setChannel(90);
  radio.setCRCLength(RF24_CRC_16);
  radio.openReadingPipe(1, pipeRX);
  radio.openWritingPipe(pipeTX);
  radio.startListening();

  sendState();
  Serial.println("RX Ready (Wall Switch Mode)");
}

/* ---------- LOOP ---------- */
void loop() {

  /* ===== WALL SWITCHES ===== */
  for (int i = 0; i < 2; i++) {
    bool reading = digitalRead(SWITCH_PIN[i]);

    if (reading != lastReading[i]) {
      lastDebounce[i] = millis();
    }

    if (millis() - lastDebounce[i] > debounceDelay) {
      if (reading != stableState[i]) {
        stableState[i] = reading;

        // LOW = switch ON (because INPUT_PULLUP)
        bool newState = (stableState[i] == LOW);

        if (loadState[i] != newState) {
          loadState[i] = newState;
          applyRelay(i);
          saveState();
          sendState();

          Serial.print("SW");
          Serial.print(i);
          Serial.print(" -> ");
          Serial.println(loadState[i] ? "ON" : "OFF");
        }
      }
    }

    lastReading[i] = reading;
  }

  /* ===== RF COMMANDS ===== */
  if (radio.available()) {
    uint8_t pkt;
    radio.read(&pkt, 1);

    uint8_t cmd = pkt >> 4;
    uint8_t d   = pkt & 0x0F;

    if (cmd == 0x1 && d < 2) {
      // Toggle from RF
      loadState[d] = !loadState[d];
    }
    else if (cmd == 0x2) {
      // All OFF
      loadState[0] = false;
      loadState[1] = false;
    }
    else if (cmd == 0x3) {
      // State request
      sendState();
      return;
    }

    applyRelay(0);
    applyRelay(1);
    saveState();
    sendState();

    Serial.println("RF command applied");
  }
}
