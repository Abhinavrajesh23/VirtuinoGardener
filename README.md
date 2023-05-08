# VirtuinoGardener 
An automatic watering system is an essential tool for any plant enthusiast who wants to keep their plants healthy and well-nourished. This system uses an ESP32 microcontroller, a soil moisture sensor, a humidity sensor, and a water pump to monitor the water content of the soil and automatically pump water when needed.


## More info
The ESP32 microcontroller is a low-cost device that is Wi-Fi enabled, making it easy to connect to the internet and access data from anywhere. The soil moisture sensor is used to measure the water content of the soil and relay this information to the microcontroller. The humidity sensor measures the relative humidity of the air, which is important for determining when plants need to be watered.

When the water content in the soil is low, the microcontroller triggers the water pump to turn on and dispense water to the plants. The system also displays the information about soil moisture, humidity, and water level on a device through the Virtuino 6 dashboard.
## things required for this build

- Esp32
- Soil Sensor
- Temperature and humidity sensor_DHT11
- -16x2 LCD Display
- Water pump-5V
- Relay Module Single-5V
- Buzzer-5V
- Power Supply-5V

## code side of things!
Note:this project is open source so anyone can copy and edit the original code!
```
#include <LiquidCrystal_I2C.h>
#include "DHT.h"
#define S1  32
#define DHTPIN 17     // Digital pin connected to the DHT sensor
#define DHTTYPE DHT11   // DHT 11
DHT dht(DHTPIN, DHTTYPE);

#ifdef ESP8266
#include <ESP8266WiFi.h>
#elif defined(ESP32)
#include <WiFi.h>
#else
#error "Board not found"
#endif
#define DEBUG_SW 1
#include <PubSubClient.h>
LiquidCrystal_I2C lcd(0x27,20,4);  // set the LCD address to 0x27 for a 16 chars and 2 line display

const int Temp_Display_Pin = 18;
int Display_Pin_State = 0; 
int Soil_Sensor_Pin = 32;   // select the input pin for the potentiometer
int Water_Pump_Relay = 25;      // select the pin for the LED
int Sensor_Value = 0;  // variable to store the value coming from the sensor
int Water_Content = 0;

int switch_ON_Flag1_previous_I = 0;
int Soil_Sensor_Value = 0;
int Soil_Sensor_Adj_Value =0;
int Soil_Sensor_Final_Value =0;
int Pump_Status = 0;

// Update these with values suitable for your network.

const char* ssid = "rajdivak_2.4GHz";
const char* password = "0559459691";
const char* mqtt_server = "192.168.1.105"; // Local IP address of Raspberry Pi

//const char* username = "HCT_ADW";
//const char* pass = "ENG@ADW";

#define sub1  "Pump_Motor/ON_OFF"
#define pub1  "Automated_Nursery/Soil_Sensor"
#define pub2  "Automated_Nursery/Temp"
#define pub3  "Automated_Nursery/Humid"
#define pub4  "Automated_Nursery/Message_L1"
#define pub5  "Automated_Nursery/Message_L2"

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;


// Connecting to WiFi Router

void setup_wifi()
{

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED)
   {
    delay(500);
    Serial.print(".");
   }
  randomSeed(micros());
  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length)
{
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  digitalWrite(Water_Pump_Relay, HIGH);

  if (strstr(topic, sub1))
  {
    for (int i = 0; i < length; i++)
    {
      Serial.print((char)payload[i]);
    }
    Serial.println();
    // Switch on the LED if an 1 was received as first character
    if ((char)payload[0] == '1')
    {
      digitalWrite(Water_Pump_Relay, HIGH);   // Turn the LED on (Note that LOW is the voltage level
      client.publish("Automated_Nursery/Message_L1", "MANUAL OPERATION MODE");
      client.publish("Automated_Nursery/Message_L2", "WATER PUMP IS ON");
      Pump_Status = 1;

    } else if ((char)payload[0] == '0')
     {
      digitalWrite(Water_Pump_Relay, LOW);  // Turn the LED off by making the voltage HIGH
      client.publish("Automated_Nursery/Message_L1", "MANUAL OPERATION MODE");
      client.publish("Automated_Nursery/Message_L2", "WATER PUMP IS OFF");
      Pump_Status = 0;
    }
    Serial.print("Pump_Status 1:");
    Serial.println(Pump_Status);
  }
  
  else
  {
    Serial.println("unsubscribed topic");
    client.publish("Automated_Nursery/Message_L2", "ALL PLANTS ARE IN GOOD CONDITION");
  }

}


// Connecting to MQTT broker

void reconnect()
{
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("Automated_Nursery/Message_L1", "WELCOME TO");
      client.publish("Automated_Nursery/Message_L2", "IoT BASED AUTOMATED AGRICULTURE NUESERY ");
      delay(4000);
      client.publish("Automated_Nursery/Message_L1", "A Project Done By");
      client.publish("Automated_Nursery/Message_L2", " ABHINAV RAJESH");
      delay(3000);
      client.publish("Automated_Nursery/Message_L1", "A Project Done By");
      client.publish("Automated_Nursery/Message_L2", "NOEL SHAIJU");
      delay(3000);
      client.publish("Automated_Nursery/Message_L1", "A Project Done By");
      client.publish("Automated_Nursery/Message_L2", "RYAN ROY");
      delay(3000);

           // ... and resubscribe
      client.subscribe(sub1);
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}



void setup()
{

Serial.println(F("DHT11 Test!....."));
  dht.begin();
  pinMode(Water_Pump_Relay, OUTPUT);
  pinMode(Soil_Sensor_Pin, INPUT);
  pinMode(Temp_Display_Pin, INPUT);
  pinMode(S1, INPUT_PULLUP);

  lcd.init();                      // initialize the lcd 
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("IoT Agriculture");
  lcd.setCursor(4,1);
  lcd.print("Nursery");

  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  delay(1000);
  
}


void loop()
{
  if (!client.connected())
  {
    reconnect();
  }
    client.loop();
    Temp_Humi();
    Manual_switching();

  if (Pump_Status == 0)
  {
    Automatic_Switching();
  }
  
}


void Manual_switching()
{
  if (digitalRead(S1) == LOW)
  {
    if (switch_ON_Flag1_previous_I == 0 )
    {
      digitalWrite(Water_Pump_Relay, LOW);
      if (DEBUG_SW) Serial.println("Water_Pump_Relay- ON");
      client.publish(pub1, "1");
      switch_ON_Flag1_previous_I = 1;
    }
  }
  if (digitalRead(S1) == HIGH )
  {
    if (switch_ON_Flag1_previous_I == 1)
    {
      digitalWrite(Water_Pump_Relay, HIGH);
      if (DEBUG_SW) Serial.println("Water_Pump_Relay OFF");
      client.publish(pub1, "0");
      switch_ON_Flag1_previous_I = 0;
    }
}
}



void Automatic_Switching()

{
  Serial.println("Automatic Switching");
  Sensor_Value = analogRead(Soil_Sensor_Pin);
  Sensor_Value = map(Sensor_Value, 1200, 2200, 0, 100);
  Water_Content = (Sensor_Value);

    Serial.print(" Water_Content_1:");
    Serial.println( Water_Content);


    if (Water_Content <= 0)
    {
    Water_Content = 0;
    }

    if (Water_Content >= 100)
    {
    Water_Content = 100;
    }

        Serial.print("Water_Content in Soil :");
        Serial.print(Water_Content);
        Serial.println( " % ");
        delay(300);


    Serial.print(" Water_Content_2:");
    Serial.println( Water_Content);
    Serial.print("Pump_Status 2:");
    Serial.println(Pump_Status);
    delay(1000);
    client.publish("Automated_Nursery/Soil_Sensor" , String(Water_Content).c_str(),true);

  //if ((Water_Content < 50) && (Pump_Status = 0))
  if (Water_Content < 50)
      {
        Serial.print("Pump Condition 1");
        digitalWrite(Water_Pump_Relay ,HIGH);
        client.publish("Automated_Nursery/Message_L1" , String("Water Pump is Started").c_str(),true);
        client.publish("Automated_Nursery/Message_L2" , String("Waterering of Plants in Progress").c_str(),true);
        delay(1000); 
      }
  if (Water_Content > 50) 
  //if  ((Water_Content > 50) && (Pump_Status = 0))
      {
        Serial.print("Pump Condition 2");
        digitalWrite(Water_Pump_Relay ,LOW);
        client.publish("Automated_Nursery/Message_L1" , String("All Plants are in ").c_str(),true);  
        client.publish("Automated_Nursery/Message_L2" , String("Good Health").c_str(),true);
        delay(1000);
      }
     // Serial.print(Pump_Status);
}



void Temp_Humi()
 {
  float h = dht.readHumidity();
  float t = dht.readTemperature();
  float f = dht.readTemperature(true);
  if (isnan(h) || isnan(t) || isnan(f))
   {
    Serial.println(F("Failed to read from DHT sensor!"));
    return;
   }
  float hif = dht.computeHeatIndex(f, h);
  float hic = dht.computeHeatIndex(t, h, false);
  Serial.print(F("Humidity: "));
  Serial.print(h);
  client.publish("Automated_Nursery/Humid" , String(h).c_str(),true); 

  delay(100);

  Serial.print(F("%  Temperature: "));
  Serial.print(t);
  Serial.println(F(" Degree C "));
  client.publish("Automated_Nursery/Temp" , String(t).c_str(),true); 
  delay(4000);

}
```
## MQTT & Virtuino dashboard
