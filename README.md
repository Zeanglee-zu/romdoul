# romdoul
#coding for control and manage wireless
#include <ESP8266WiFi.h>
#include <Adafruit_NeoPixel.h>
#include <ESP8266HTTPClient.h>
#include <ESP8266mDNS.h>
#include <WiFiUdp.h>
#include <ArduinoOTA.h>
#include<WiFiManager.h>
#include<ESP8266WebServer.h>
#include<EEPROM.h>
#include <Servo.h>
#include <ArduinoJson.h>

#define first

#define blow D7
#define PIN D8
#define NUMPIXELS 32
#define BRIGHTNESS 50
const size_t capacity = JSON_OBJECT_SIZE(10) + 400;
DynamicJsonDocument doc(capacity);

Servo servo;

String url = "";
String id = "";
unsigned long count = 0;
unsigned long timeoff = 0;
const int idadd = 0;
char strget[30];

HTTPClient http;
unsigned long timer;

Adafruit_NeoPixel pixels(NUMPIXELS, PIN, NEO_GRB + NEO_KHZ800); // RGB
boolean leden = true;

void setup() {
  servo.attach(D6, 600, 2000); //600,2200
  pinMode(blow, OUTPUT);
  digitalWrite(blow, 0);
  servo.write(180);
  Serial.begin(9600);
  EEPROM.begin(512);
  WiFiManager wifiManager;
  wifiManager.autoConnect("Romdoul", "12345678");

  pixels.begin();
  while (WiFi.status() != WL_CONNECTED) {
    connecting();
  }
  ///////////////////////////RGB/////////////////////////
  for (int j = 0; j < NUMPIXELS; j++) {
    pixels.setPixelColor(j, pixels.Color(0, 0, 255));
  }
  pixels.setBrightness(100);
  pixels.show();
  /////////////////////////////////////////////////////
  ArduinoOTA.setPort(8266);
  ArduinoOTA.setHostname("Romdoul");
  ArduinoOTA.begin();
  timer = millis();

#ifdef first
  String idstr = String(__DATE__) + String(__TIME__);
  idstr.replace(" ", "");
  idstr.replace(":", "");
  //idstr = "Jan20200202152101";
  Serial.println("ID = " + idstr);
  for (int m = 0; m <= 29; m++) {
    strget[m] = 0;
  }
  strcpy(strget, idstr.c_str());
  EEPROM.begin(512);
  EEPROM.put(idadd, strget);
  delay(10);
  EEPROM.end();
#endif
}

void loop() {
  ArduinoOTA.handle();
  if (millis() - timer > 1000) {
    count++;
    for (int m = 0; m <= 29; m++) {
      strget[m] = 0;
    }
    EEPROM.begin(512);
    EEPROM.get(idadd, strget);
    EEPROM.commit();
    EEPROM.end();
    id = String(strget);
    //Serial.println(id);
    http.begin(url + "?devices_id=" + id);
    int httpcode = http.GET();
    if (httpcode > 0) {
      String res = http.getString();
      //Serial.println(res);
      if (res.indexOf("ID not correct") < 0) {
        res = res.substring(1, res.length() - 1);
        Serial.println(res);
        http.end();
        DeserializationError error = deserializeJson(doc, res);
        if (error) {
          Serial.print(F("deserializeJson() failed: "));
          Serial.println(error.c_str());
          return;
        }
        String s1 = doc["smell1"].as<char*>();
        String s2 = doc["smell2"].as<char*>();
        String s3 = doc["smell3"].as<char*>();
        String light = doc["light_status"].as<char*>();
        String light_color = doc["light_color"].as<char*>();
        String dur = doc["duration"].as<char*>();
        if ( timeoff != dur.toInt()) {
          timeoff = dur.toInt();
          count = 0;
        }
        if ((s1.indexOf("OFF") > -1 ) && (s2.indexOf("OFF") > -1 ) && (s3.indexOf("OFF") > -1 )) {
          servo.write(180);
          digitalWrite(blow, 0);
        }
        if ((s1.indexOf("ON") > -1 ) && (s2.indexOf("OFF") > -1 ) && (s3.indexOf("OFF") > -1 )) {
          servo.write(0);
          digitalWrite(blow, 1);
          count = 0;
        }
        if ((s1.indexOf("OFF") > -1 ) && (s2.indexOf("ON") > -1 ) && (s3.indexOf("OFF") > -1 )) {
          servo.write(60);
          digitalWrite(blow, 1);
          count = 0;
        }
        if ((s1.indexOf("OFF") > -1 ) && (s2.indexOf("OFF") > -1 ) && (s3.indexOf("ON") > -1 )) {
          servo.write(125);
          digitalWrite(blow, 1);
          count = 0;
        }
        if (light.indexOf("OFF") > -1) {
          leden = false;
          pixels.setBrightness(0);
          pixels.show();
        } else {
          leden = true;
          pixels.setBrightness(100);
          pixels.show();
        }
        if (leden == true) {
          String strr = light_color.substring(3, 5);
          String strg = light_color.substring(5, 7);
          String strb = light_color.substring(7);
          int red = strtol(strr.c_str(), NULL, 16);
          int green = strtol(strg.c_str(), NULL, 16);
          int blue = strtol(strb.c_str(), NULL, 16);
          pixels.setBrightness(100);
          for (int j = 0; j < NUMPIXELS; j++) {
            pixels.setPixelColor(j, pixels.Color(red, green, blue));
          }
          pixels.show();
        }
      }
    }
    if (timeoff > 0) {
      if (count >= timeoff * 60) {
        servo.write(180);
        digitalWrite(blow, 0);
      }
    }
    timer = millis();
  }
}

void connecting() {
  for (int i = 0; i < 100; i++) {
    for (int j = 0; j < NUMPIXELS; j++) {
      pixels.setPixelColor(j, pixels.Color(255, 0, 0));
    }
    pixels.setBrightness(i);
    pixels.show();
    delay(10);
  }
  for (int i = 100; i > 0; i--) {
    for (int j = 0; j < NUMPIXELS; j++) {
      pixels.setPixelColor(j, pixels.Color(255, 0, 0));
    }
    pixels.setBrightness(i);
    pixels.show();
    delay(10);
  }
}

