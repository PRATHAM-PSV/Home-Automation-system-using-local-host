#include <WiFi.h>
#include <WebServer.h>
#include <Preferences.h>
#include <SPI.h>
#include <RF24.h>

/* ---------- WIFI AP ---------- */
const char* AP_SSID = "Home";
const char* AP_PASS = "Anita2005";

/* ---------- LOAD CONFIG ---------- */
#define LOCAL 4
#define REMOTE 2
#define TOTAL 6

uint8_t RELAY_PIN[LOCAL]   = {2, 4, 25, 26};
uint8_t SWITCH_PIN[LOCAL] = {13, 12, 14, 27};

bool loadState[TOTAL];

/* ---------- WEB ---------- */
WebServer server(80);
Preferences prefs;

/* ---------- RF ---------- */
#define CE_PIN 21
#define CSN_PIN 22
#define SCK_PIN 18
#define MOSI_PIN 23
#define MISO_PIN 19

RF24 radio(CE_PIN, CSN_PIN);
const byte pipeTX[6] = "TXA01";
const byte pipeRX[6] = "RXA01";

/* ---------- RELAY ---------- */
void applyRelay(uint8_t i) {
  // Active-LOW relay module
  digitalWrite(RELAY_PIN[i], loadState[i] ? LOW : HIGH);
}

/* ---------- LOCAL WALL SWITCHES ---------- */
void readLocalSwitches() {
  static bool lastReading[LOCAL]  = {HIGH, HIGH, HIGH, HIGH};
  static bool stableState[LOCAL]  = {HIGH, HIGH, HIGH, HIGH};
  static unsigned long lastDebounce[LOCAL] = {0};
  const unsigned long debounceDelay = 50;

  for (int i = 0; i < LOCAL; i++) {
    bool reading = digitalRead(SWITCH_PIN[i]);

    if (reading != lastReading[i]) {
      lastDebounce[i] = millis();
    }

    if ((millis() - lastDebounce[i]) > debounceDelay) {
      if (reading != stableState[i]) {
        stableState[i] = reading;

        // LOW = switch ON (because INPUT_PULLUP)
        bool newState = (stableState[i] == LOW);

        if (loadState[i] != newState) {
          loadState[i] = newState;
          applyRelay(i);
          prefs.putBool(("L" + String(i)).c_str(), loadState[i]);

          Serial.print("SW");
          Serial.print(i);
          Serial.print(" -> ");
          Serial.println(loadState[i] ? "ON" : "OFF");
        }
      }
    }

    lastReading[i] = reading;
  }
}

/* ---------- RF SEND ---------- */
void rfSend(uint8_t cmd, uint8_t data) {
  uint8_t pkt = (cmd << 4) | (data & 0x0F);
  radio.stopListening();
  radio.write(&pkt, 1);
  radio.startListening();
}

/* ---------- RF READ ---------- */
void rfRead() {
  while (radio.available()) {
    uint8_t pkt;
    radio.read(&pkt, 1);

    uint8_t cmd = pkt >> 4;
    uint8_t d   = pkt & 0x0F;

    if (cmd == 0x8) {   // STATE RESPONSE FROM REMOTE
      loadState[4] = d & 0x01;
      loadState[5] = (d & 0x02) >> 1;
    }
  }
}

/* ---------- JSON ---------- */
String jsonState() {
  String j = "{";
  for (int i = 0; i < TOTAL; i++) {
    j += "\"L" + String(i) + "\":" + (loadState[i] ? "true" : "false");
    if (i < TOTAL - 1) j += ",";
  }
  j += "}";
  return j;
}

/* ---------- WEB ---------- */
void handleState() {
  server.send(200, "application/json", jsonState());
}

void handleToggle() {
  if (!server.hasArg("i")) {
    server.send(400, "text/plain", "Missing i");
    return;
  }

  int i = server.arg("i").toInt();
  if (i < 0 || i >= TOTAL) {
    server.send(400, "text/plain", "Invalid i");
    return;
  }

  if (i < LOCAL) {
    // Web toggle for LOCAL load
    loadState[i] = !loadState[i];
    applyRelay(i);
    prefs.putBool(("L" + String(i)).c_str(), loadState[i]);
  } else {
    // Remote toggle
    rfSend(0x1, i - LOCAL);
  }

  server.send(200, "application/json", jsonState());
}

void handleAllOff() {
  for (int i = 0; i < LOCAL; i++) {
    loadState[i] = false;
    applyRelay(i);
    prefs.putBool(("L" + String(i)).c_str(), false);
  }

  rfSend(0x2, 0);   // ALL OFF REMOTE
  server.send(200, "text/plain", "OK");
}

/* ---------- MOBILE UI ---------- */
void handleRoot() {
  server.send(200, "text/html", R"HTML(
<!DOCTYPE html><html>
<head>
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Home Control</title>
<style>
body{margin:0;background:#0b0e13;color:#fff;font-family:system-ui}
.header{padding:16px;background:#121622;text-align:center;font-size:20px}
.section{margin:18px 14px 6px;color:#aaa}
.grid{display:grid;grid-template-columns:1fr 1fr;gap:14px;padding:0 14px}
.card{background:#141a2a;border-radius:18px;padding:18px;text-align:center}
.name{margin-bottom:14px}
.btn{width:88px;height:88px;margin:auto;border-radius:50%;
display:flex;align-items:center;justify-content:center;
background:#1f2436;color:#777;font-weight:600}
.btn.on{background:#3cff57;color:#000;box-shadow:0 0 20px #3cff57}
.footer{padding:16px}
.alloff{width:100%;height:64px;border-radius:32px;
border:none;font-size:18px;background:#ff3c3c;color:#fff}
</style>
</head>
<body>
<div class="header">Home Control</div>

<div class="section">Main House</div>
<div class="grid">
<div class="card"><div class="name">Light 1</div><div class="btn" id="b0" onclick="tg(0)">OFF</div></div>
<div class="card"><div class="name">Light 2</div><div class="btn" id="b1" onclick="tg(1)">OFF</div></div>
<div class="card"><div class="name">Socket</div><div class="btn" id="b2" onclick="tg(2)">OFF</div></div>
<div class="card"><div class="name">Fan</div><div class="btn" id="b3" onclick="tg(3)">OFF</div></div>
</div>

<div class="section">Second House</div>
<div class="grid">
<div class="card"><div class="name">Remote 1</div><div class="btn" id="b4" onclick="tg(4)">OFF</div></div>
<div class="card"><div class="name">Remote 2</div><div class="btn" id="b5" onclick="tg(5)">OFF</div></div>
</div>

<div class="footer">
<button class="alloff" onclick="allOff()">ALL OFF</button>
</div>

<script>
const N=6;
function upd(s){
  for(let i=0;i<N;i++){
    let b=document.getElementById("b"+i);
    if(s["L"+i]){b.classList.add("on");b.innerText="ON";}
    else{b.classList.remove("on");b.innerText="OFF";}
  }
}
function poll(){fetch("/state").then(r=>r.json()).then(upd);setTimeout(poll,500);}
function tg(i){fetch("/toggle?i="+i);}
function allOff(){fetch("/alloff");}
poll();
</script>
</body></html>
)HTML");
}

/* ---------- SETUP ---------- */
void setup() {
  Serial.begin(115200);

  for (int i = 0; i < LOCAL; i++) {
    pinMode(RELAY_PIN[i], OUTPUT);
    pinMode(SWITCH_PIN[i], INPUT_PULLUP);
  }

  prefs.begin("home", false);

  // Load saved states
  for (int i = 0; i < LOCAL; i++) {
    loadState[i] = prefs.getBool(("L" + String(i)).c_str(), false);
    applyRelay(i);
  }

  WiFi.softAP(AP_SSID, AP_PASS);
  Serial.print("AP IP: ");
  Serial.println(WiFi.softAPIP());

  server.on("/", handleRoot);
  server.on("/state", handleState);
  server.on("/toggle", handleToggle);
  server.on("/alloff", handleAllOff);
  server.begin();

  SPI.begin(SCK_PIN, MISO_PIN, MOSI_PIN, CSN_PIN);
  radio.begin();
  radio.setPALevel(RF24_PA_HIGH);
  radio.setDataRate(RF24_250KBPS);
  radio.setChannel(90);
  radio.setCRCLength(RF24_CRC_16);
  radio.openWritingPipe(pipeTX);
  radio.openReadingPipe(1, pipeRX);
  radio.startListening();

  rfSend(0x3, 0); // REQUEST STATE
  Serial.println("TX Ready (Wall Switch Mode)");
}

void loop() {
  server.handleClient();
  rfRead();
  readLocalSwitches();
}
