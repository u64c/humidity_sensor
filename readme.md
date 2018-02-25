/*********************************************************************
This is an example for our Monochrome OLEDs based on SSD1306 drivers

  Pick one up today in the adafruit shop!
  ------> http://www.adafruit.com/category/63_98

This example is for a 128x64 size display using I2C to communicate
3 pins are required to interface (2 I2C and one reset)

Adafruit invests time and resources providing this open source code, 
please support Adafruit and open-source hardware by purchasing 
products from Adafruit!

Written by Limor Fried/Ladyada  for Adafruit Industries.  
BSD license, check license.txt for more information
All text above, and the splash screen must be included in any redistribution
*********************************************************************/
#include <ESP8266WiFi.h>
#include <SPI.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "PietteTech_DHT.h"
#define DHTTYPE  DHT22           // Sensor type DHT11/21/22/AM2301/AM2302
#define DHTPIN   D6              // Digital pin for communications

#define REPORT_INTERVAL 5000 // in msec must > 2000

#define OLED_RESET BUILTIN_LED
Adafruit_SSD1306 display(OLED_RESET);

#define VENTI_ON D7

#if (SSD1306_LCDHEIGHT != 64)
#error("Height incorrect, please fix Adafruit_SSD1306.h!");
#endif

//extern "C" {
//#include "user_interface.h"
//}
//os_timer_t myTimer;

// read analogport
int var = 0;
int thresholdHumidity;
const int analogInPin = A0;

// to check dht
unsigned long startMills;
float t, h, d;
int acquireresult;
int acquirestatus;
bool state;
unsigned long finalTime = 3600000;
unsigned long activeTime = 1200000;
//unsigned long finalTime = 10000;
//unsigned long activeTime = 5000;
unsigned long startTime;
bool timeFlag = false;



//declaration
void dht_wrapper(); // must be declared before the lib initialization

// Lib instantiate
PietteTech_DHT DHT(DHTPIN, DHTTYPE, dht_wrapper);

// globals
bool bDHTstarted;       // flag to indicate we started acquisition

// This wrapper is in charge of calling
// must be defined like this for the lib work
void dht_wrapper() {
  DHT.isrCallback();
}

void setup()   {  
  startMills = millis();
  
  Serial.begin(115200);

  /*
    while (!Serial) {
    yield(); // wait for serial port to connect.
    }
  */

  Serial.println("");
  Serial.println("DHT Example program using DHT.acquire and DHT.acquireAndWait");
  Serial.println("");

  // delay 2 sec before start acquire
  while ( (millis() - startMills) < 2000 ) {
    yield();
  }

  // blocking method
  acquirestatus = 0;
  acquireresult = DHT.acquireAndWait(1000);
  if ( acquireresult == 0 ) {
    t = DHT.getCelsius();
    h = DHT.getHumidity();
    d = DHT.getDewPoint();
  } else {
    t = h = d = 0;
  }
               
  pinMode(VENTI_ON, OUTPUT);

  // by default, we'll generate the high voltage from the 3.3v line internally! (neat!)
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);  // initialize with the I2C addr 0x3D (for the 128x64)
  // init done
  display.clearDisplay();
  display.setTextColor(WHITE);
  display.setTextSize(2.5);
}


void loop() {
  Serial.print("Startzeit: ");
  Serial.println(startTime);
  Serial.print("Aktuelle Zeit: ");
  Serial.println(millis());
  
    
  setupDHT();
  
  writeDisplay();
  
  createList();
  
  setup_output();  
  //Serial.println(analogRead(A0));
  delay(200);
}

void setup_output() 
{
  //if (time_now() - start_time < 2) set_output();
  //else clear_output();
  //if (time_now() - start_time > 30*60) start_time+=30*60;

  //Serial.print(h);
 
     
    unsigned long nowTime = millis(); 

    if (timeFlag == false)
    {
    Serial.println("Set Starttime: ");
    startTime = nowTime;
    }
    
    if (h > thresholdHumidity) 
    {
      timeFlag = true;
      if (nowTime - startTime < finalTime)
      {
        if (nowTime - startTime < activeTime)
        {
        digitalWrite(VENTI_ON, LOW);
        state = true;
        Serial.println("State 1 1");
        }
        else
        {
        digitalWrite(VENTI_ON, HIGH);
        state = false;
        Serial.println("State 0 1");
        }
      }
      else
      {
      timeFlag = false;
      }
    }

    
    
    if (h <= thresholdHumidity) 
    {
      timeFlag = false;
      digitalWrite(VENTI_ON, HIGH);
      state = false;
      Serial.println("State 0 0");
      delay(100);
    }
}

void createList ()
  {
    int offset = 49;
    int maxFor = 51;
    int preValues[51];
    for (int i = 0; i < maxFor; i++){
      preValues[i] = 20 * i;
      //Serial.println(i);
      //Serial.print("preValues: ");
      //Serial.println(preValues[maxFor]);
      if (analogRead(analogInPin) < preValues[i])
      {
        thresholdHumidity = i + offset;
        //Serial.print("Schwellwert: ");        
        //Serial.println(thresholdHumidity);
        break;
      }
      
      if ((i==(maxFor -1))&&(analogRead(analogInPin) >= preValues[(maxFor-1)]))
      {
        thresholdHumidity = maxFor + offset;
        //Serial.print("Schwellwertmax: ");        
        //Serial.println(thresholdHumidity);
        break;
      }
      
    }
  }

void setupDHT() 
  {
    if (bDHTstarted) {
    acquirestatus = DHT.acquiring();
    if (!acquirestatus) {
      acquireresult = DHT.getStatus();
      if ( acquireresult == 0 ) {
        t = DHT.getCelsius();
        h = DHT.getHumidity();
        d = DHT.getDewPoint();
      }
      bDHTstarted = false;
    }
  }



  if ((millis() - startMills) > REPORT_INTERVAL) {
    if ( acquireresult == 0 ) {
        

    } else {
      Serial.println("Is dht22 connected");
    }
    startMills = millis();

    // to remove lock
    if (acquirestatus == 1) {
      DHT.reset();
    }

    if (!bDHTstarted) {
      // non blocking method
      DHT.acquire();
      bDHTstarted = true;
    }
  }
  }

void writeDisplay()
  {
  display.clearDisplay();
  display.setCursor(0,0);
  display.print("T: ");
  display.print(t);
  display.println(" C");
  display.print("F: ");
  display.print(h);
  display.println(" %");
  display.print("SW: ");
  display.print(thresholdHumidity);
  display.println(" %");
  display.print("VENTI: ");
  display.print(state);
  display.display();
  }
