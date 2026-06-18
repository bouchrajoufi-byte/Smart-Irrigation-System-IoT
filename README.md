Annex A : Programme complet de arduino

#include <DHT.h>
#include <OneWire.h>
#include <DallasTemperature.h>
int flameSensor = A3;
int soilSensor  = A2;
int rainSensor  = A1;
int gasSensor   = A0;
int waterSensor = A4;
#define RELAIS_VENT    2
#define RELAIS_POMPE   3
#define RELAIS_BUZZER  5
#define RELAIS_4       6
#define RELAIS_5       7
#define DHTPIN 4
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);
#define ONE_WIRE_BUS 8
OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature ds18b20(&oneWire);
int flameValue;
int soilValue;
int rainValue;
int gasValue;
int waterValue;
float tempDHT;
float humDHT;
float tempDS;
bool autoMode_vent   = true;
bool autoMode_pompe  = true;
bool autoMode_buzzer = true;
unsigned long lastSend = 0;
const unsigned long INTERVAL = 2000;
void setup() {
  pinMode(RELAIS_VENT, OUTPUT);
  pinMode(RELAIS_POMPE, OUTPUT);
  pinMode(RELAIS_BUZZER, OUTPUT);
  pinMode(RELAIS_4, OUTPUT);
  pinMode(RELAIS_5, OUTPUT);
  digitalWrite(RELAIS_VENT, HIGH);
  digitalWrite(RELAIS_POMPE, HIGH);
  digitalWrite(RELAIS_BUZZER, HIGH);
  digitalWrite(RELAIS_4, HIGH);
  digitalWrite(RELAIS_5, HIGH);
  Serial.begin(9600);
  dht.begin();
  ds18b20.begin();
  Serial.println("SMART FARM READY");
}
void loop() {
  lireCommandes();
  if (millis() - lastSend >= INTERVAL) {
    lastSend = millis();
    flameValue = analogRead(flameSensor);
    soilValue  = analogRead(soilSensor);
    rainValue  = analogRead(rainSensor);
    gasValue   = analogRead(gasSensor);
    waterValue = analogRead(waterSensor);
    tempDHT = dht.readTemperature();
    humDHT  = dht.readHumidity();
    if (isnan(tempDHT)) tempDHT = 0;
    if (isnan(humDHT)) humDHT = 0;
    ds18b20.requestTemperatures();
    tempDS = ds18b20.getTempCByIndex(0);
    if (tempDS == -127.0)
      tempDS = 0;
    bool flame = false;
    if (flameValue < 150) {
      flame = true;
    }
    bool rain = false;
    if (rainValue < 500) {
      rain = true;
    }
    bool soilDry = false;
    if (soilValue > 700) {
      soilDry = true;
    }
    bool gas = false;
    if (gasValue > 500) {
      gas = true;
    }
    bool waterHigh = false;
    if (waterValue > 750) {
      waterHigh = true;
    }
    if (autoMode_vent) {
      if (tempDHT >= 32 || gas) {
        digitalWrite(RELAIS_VENT, LOW);
      } else {
        digitalWrite(RELAIS_VENT, HIGH);
      }
    }
    if (autoMode_pompe) {
      if (soilDry && !rain) {
        digitalWrite(RELAIS_POMPE, LOW);
      } else {
        digitalWrite(RELAIS_POMPE, HIGH);
      }
    }
    if (autoMode_buzzer) {
      if (flame || gas || waterHigh) {
        digitalWrite(RELAIS_BUZZER, LOW);
      } else {
        digitalWrite(RELAIS_BUZZER, HIGH);
      }
    }
    bool vent_on   = (digitalRead(RELAIS_VENT) == LOW);
    bool pompe_on  = (digitalRead(RELAIS_POMPE) == LOW);
    bool buzzer_on = (digitalRead(RELAIS_BUZZER) == LOW);

    Serial.println("\n========= SMART FARM =========");
    Serial.print("FLAMME : ");
    Serial.println(flame ? "DETECTEE" : "NON");
    Serial.print("PLUIE : ");
    Serial.println(rain ? "OUI" : "NON");
    Serial.print("SOL : ");
    Serial.println(soilDry ? "SEC" : "HUMIDE");
    Serial.print("GAZ : ");
    Serial.println(gas ? "DANGEREUX" : "OK");
    Serial.print("EAU : ");
    Serial.println(waterHigh ? "HAUT" : "NORMAL");
    Serial.print("DHT11 : ");
    Serial.print(tempDHT);
    Serial.print(" C | ");
    Serial.print(humDHT);
    Serial.println(" %");
    Serial.print("DS18B20 : ");
    Serial.print(tempDS);
    Serial.println(" C");
    Serial.print("VENTILATEUR : ");
    Serial.println(vent_on ? "ON" : "OFF");
    Serial.print("POMPE : ");
    Serial.println(pompe_on ? "ON" : "OFF");
    Serial.print("BUZZER : ");
    Serial.println(buzzer_on ? "ON" : "OFF");
    Serial.println("================================");
    Serial.print("{");
    Serial.print("\"temp\":");
    Serial.print(tempDHT, 1);
    Serial.print(",");
    Serial.print("\"hum\":");
    Serial.print(humDHT, 1);
    Serial.print(",");
    Serial.print("\"temp2\":");
    Serial.print(tempDS, 1);
    Serial.print(",");
    Serial.print("\"gaz\":");
    Serial.print(gasValue);
    Serial.print(",");
    Serial.print("\"flamme\":");
    Serial.print(flameValue);
    Serial.print(",");
    Serial.print("\"pluie\":");
    Serial.print(rainValue);
    Serial.print(",");
    Serial.print("\"sol\":");
    Serial.print(soilValue);
    Serial.print(",");
    Serial.print("\"eau\":");
    Serial.print(waterValue);
    Serial.print(",");
    Serial.print("\"vent\":");
    Serial.print(vent_on ? 1 : 0);
    Serial.print(",");
    Serial.print("\"pompe\":");
    Serial.print(pompe_on ? 1 : 0);
    Serial.print(",");
    Serial.print("\"buzzer\":");
    Serial.print(buzzer_on ? 1 : 0);
    Serial.println("}");
  }
}
void lireCommandes() {
  if (Serial.available() > 0) {
    String cmd = Serial.readStringUntil('\n');
    cmd.trim();
    if (cmd.startsWith("R")) {
      int relais = cmd.charAt(1) - '0';
      int etat   = cmd.charAt(3) - '0';
      int pin;
      switch (relais) {
        case 1: pin = RELAIS_VENT; break;
        case 2: pin = RELAIS_POMPE; break;
        case 3: pin = RELAIS_BUZZER; break;
        case 4: pin = RELAIS_4; break;
        case 5: pin = RELAIS_5; break;
        default: return;
      }
      digitalWrite(pin, etat == 1 ? LOW : HIGH);
      Serial.print("{\"cmd\":\"ok\",\"relais\":");
      Serial.print(relais);
      Serial.print(",\"etat\":");
      Serial.print(etat);
      Serial.println("}");
    }
    else if (cmd.startsWith("A")) {
      int num = cmd.charAt(1) - '0';
      bool mode = (cmd.charAt(3) == '1');
      if (num == 1) autoMode_vent = mode;
      if (num == 2) autoMode_pompe = mode;
      if (num == 3) autoMode_buzzer = mode;
      Serial.println("{\"cmd\":\"auto_ok\"}");
    }
  }
}


