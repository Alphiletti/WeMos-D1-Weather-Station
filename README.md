# WeMos D1 R1 - Weather Station

ESP8266 based weather station. It request from the [OpenWeather](https://openweathermap.org/) API and getting data from there. Also DHT11 sensor measures room temperature and humidity.  All data is shown on 2x16 LCD screen and stored in microSD card.

<u>**Equipments:**</u>

- WeMos D1 R1
- 2x16 LCD Screen with LCD I2C Serial Interface Module
- DHT11 Temperature and Humidity Module (built-in 10k pull-up resistor)
- DS3231 Precision RTC Module
- MicroSD Card Module
- Breadboard
- Jumper Wires

**<u>Schematic:</u>**

[imgur](https://imgur.com/a/DNQjxUj)

![Schmatic](images\sketch.png)


​    

**<u>Images</u>**

![](images\img1.png)

<img src="images\gif2.gif" width="360" height="304">

![](images\log.png)

<u>**Code**</u>

```c++
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include "DHT.h"
#include <ESPWiFi.h>
#include <ESPHTTPClient.h>
#include <JsonListener.h>
#include <ESP8266WiFi.h>
#include <ArduinoJson.h>
#include <DS3231.h>
#include <SPI.h>
#include "SdFat.h"


LiquidCrystal_I2C lcd(0x27, 16, 2); // set the LCD address to 0x27 for a 16 chars and 2 line display
#define DHTPIN D1
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

const char* ssid     = "ALPEREN";      // SSID of local network
const char* password = "xxxxxxxxxxxx";   // Password on network
String apiKey = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"; // from openweathermap.org
String cityID = "323786";

/*
İstanbul - 745042
İzmir - 311046
Ankara - 323786
*/
WiFiClient client;
char servername[] = "api.openweathermap.org";
String result;

int  counter = 60;

String weatherDescription = "";
String weatherLocation = "";
String Country;
float Temperature;
float Humidity;
float Pressure;
float DHT_Humidity;
float DHT_Celcius;
int DayValue;

RTClib myRTC;
DS3231 Clock;

using namespace sdfat;
SdFat SD;

#define SD_CS_PIN SS

void setup()
{
  Serial.begin(57600);
  Wire.begin();
  delay(500);

  lcd.init();
  lcd.backlight();
  int cursorPosition = 0;
  lcd.begin(16, 2);
  lcd.print("   Connecting");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    lcd.setCursor(cursorPosition, 1);
    lcd.print(".");
    cursorPosition++;
  }
  lcd.clear();
  lcd.print("   Connected!");
  delay(1000);
  lcd.clear();
}


void loop()
{
  dht.begin();
  //Site verileri
  if (counter == 60)
  {
    getWeatherData();
    counter = 0;
    delay(2000);
    displayWeather(weatherLocation, weatherDescription, counter);
    delay(5000);
    displayConditions(Temperature, Humidity, Pressure);
    delay(5000);
  }
  else
  {
    counter++;
    displayWeather(weatherLocation, weatherDescription, counter);
    delay(5000);
    displayConditions(Temperature, Humidity, Pressure);
    delay(5000);
  }
  lcd.clear();

  getDhtData();
  lcd.clear();

  //RTC Saat
  int i = 0;
  while (i < 5) {
    rtcPage();
    i++;
  }
  lcd.clear();
  logToSD(Temperature, Humidity, Pressure, weatherLocation, weatherDescription, Country, DHT_Humidity, DHT_Celcius,DayValue);
}

void getWeatherData()
{
  lcd.setCursor(0, 0);
  lcd.print("Data");
  lcd.setCursor(0, 1);
  lcd.print("Updating");

  if (client.connect(servername, 80)) {  //starts client connection, checks for connection
    client.println("GET /data/2.5/weather?id=" + cityID + "&units=metric&APPID=" + apiKey);
    client.println("Host: api.openweathermap.org");
    client.println("User-Agent: ArduinoWiFi/1.1");
    client.println("Connection: close");
    client.println();
  }
  else {
    Serial.println("connection failed"); //error message if no client connect
    Serial.println();
  }

  while (client.connected() && !client.available()) delay(1); //waits for data
  while (client.connected() || client.available()) { //connected or data available
    char c = client.read(); //gets byte from ethernet buffer
    result = result + c;
  }

  client.stop(); //stop client
  result.replace('[', ' ');
  result.replace(']', ' ');
  Serial.println(result);

  char jsonArray [result.length() + 1];
  result.toCharArray(jsonArray, sizeof(jsonArray));
  jsonArray[result.length() + 1] = '\0';

  StaticJsonBuffer<1024> json_buf;
  JsonObject &root = json_buf.parseObject(jsonArray);
  if (!root.success())
  {
    Serial.println("parseObject() failed");
  }

  String location = root["name"];
  String country = root["sys"]["country"];
  float temperature = root["main"]["temp"];
  float humidity = root["main"]["humidity"];
  String weather = root["weather"]["main"];
  String description = root["weather"]["description"];
  float pressure = root["main"]["pressure"];

  weatherDescription = description;
  weatherLocation = location;
  Country = country;
  Temperature = temperature;
  Humidity = humidity;
  Pressure = pressure;

}

void getDhtData()
{
  float dht_humidity = dht.readHumidity();
  float dht_celsius = dht.readTemperature();
  lcd.setCursor(2, 0);
  lcd.print("Temp");
  lcd.setCursor(1, 1);
  lcd.print(dht_celsius);
  lcd.print((char)223);
  lcd.print("C");
  lcd.setCursor(11, 0);
  lcd.print("Hum");
  lcd.setCursor(10, 1);
  lcd.print(dht_humidity);
  DHT_Humidity = dht_humidity;
  DHT_Celcius = dht_celsius;
  delay(5000);
}

void displayWeather(String location, String description, int counter)
{
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print(location);
  lcd.print(", ");
  lcd.print(Country);
  lcd.setCursor(0, 1);
  lcd.print(description);
  lcd.setCursor(14, 1);
  lcd.print(counter);
}

void displayConditions(float Temperature, float Humidity, float Pressure)
{
  lcd.clear();
  lcd.print(" T:");
  lcd.print(Temperature, 1);
  lcd.print((char)223);
  lcd.print("C ");
  lcd.print(" H:");
  lcd.print(Humidity, 0);
  lcd.print("%");
  lcd.setCursor(0, 1);
  lcd.print("  P:");
  lcd.print(Pressure, 1);
  lcd.print(" hPa");
}

void rtcPage()
{
  DateTime now = myRTC.now();
  char daysOfTheWeek[7][4] = {"Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"};
  int dayValue = Clock.getDoW();
  String dayofweek = daysOfTheWeek[dayValue];
  lcd.setCursor(1, 0);
  if (now.day() < 10)
  {
    lcd.print("0");
    lcd.print(now.day(), DEC);
  }
  else
  {
    lcd.print(now.day(), DEC);
  }
  lcd.print('/');
  if (now.month() < 10)
  {
    lcd.print("0");
    lcd.print(now.month(), DEC);
  }
  else
  {
    lcd.print(now.month(), DEC);
  }
  lcd.print('/');
  lcd.print(now.year(), DEC);
  lcd.print(' ');
  lcd.print(dayofweek);
  lcd.setCursor(4, 1);
  if (now.hour() < 10)
  {
    lcd.print("0");
    lcd.print(now.hour(), DEC);
  }
  else
  {
    lcd.print(now.hour(), DEC);
  }
  lcd.print(':');
  if (now.minute() < 10)
  {
    lcd.print("0");
    lcd.print(now.minute(), DEC);
  }
  else
  {
    lcd.print(now.minute(), DEC);
  }
  lcd.print(':');
  if (now.second() < 10)
  {
    lcd.print("0");
    lcd.print(now.second(), DEC);
  }
  else
  {
    lcd.print(now.second(), DEC);
  }
  DayValue = dayValue;
  delay(1000);
}

void logToSD(float Temperature, float Humidity, float Pressure, String weatherLocation, String weatherDescription, String Country, float dht_humidity, float dht_celsius, int DayValue)
{
  lcd.setCursor(5, 0);
  lcd.print("Saving");
  delay(1000);
  sdfat::File myFile;
  while (!Serial) {
    ; // wait for serial port to connect. Needed for native USB port only
  }

  Serial.print("Initializing SD card...");

  if (!SD.begin(SD_CS_PIN)) {
    Serial.println("initialization failed!");
    return;
  }
  Serial.println("initialization done.");

  // open the file. note that only one file can be open at a time,
  // so you have to close this one before opening another.
  myFile = SD.open("weather8.txt", FILE_WRITE);

  // if the file opened okay, write to it:
  if (myFile) {
    Serial.print("Writing to weather8.txt...");
    
    DateTime nowx = myRTC.now();
    char fullDayNames[7][10] = {"Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"};
    String fullName = fullDayNames[DayValue];
    myFile.print("[");
    myFile.print(nowx.day());
    myFile.print("/");
    myFile.print(nowx.month());
    myFile.print("/");
    myFile.print(nowx.year());
    myFile.print(" ");
    myFile.print(fullName);
    myFile.print(" ");
    myFile.print(nowx.hour());
    myFile.print(":");
    myFile.print(nowx.minute());
    myFile.print(":");
    myFile.print(nowx.second());
    myFile.println("]");
    myFile.println("");
    myFile.print("[");
    myFile.print("API");
    myFile.println("]");
    myFile.print(Country);
    myFile.print(" ");
    myFile.print(weatherLocation);
    myFile.print(" - ");
    myFile.println(weatherDescription);
    myFile.print("Temperature:");
    myFile.print(Temperature, 1);
    myFile.print((char)176);
    myFile.print("C ");
    myFile.print(" Humidity:");
    myFile.print(Humidity, 0);
    myFile.print("%");
    myFile.print("  Pressure:");
    myFile.print(Pressure, 1);
    myFile.println(" hPa");
    myFile.println("");
    myFile.print("[");
    myFile.print("DHT");
    myFile.println("]");
    myFile.print("DHT Temperature:");
    myFile.print(dht_celsius);
    myFile.print((char)176);
    myFile.print("C");
    myFile.print("  DHT Humidity:");
    myFile.print("%");
    myFile.println(dht_humidity);
    myFile.println("------------------------------------------------------");

    // close the file:
    myFile.close();
    Serial.println("done.");
    } 
    else {
    // if the file didn't open, print an error:
    Serial.println("error opening weather8.txt");
  }

  // re-open the file for reading:
  myFile = SD.open("weather8.txt");
  if (myFile) {
    Serial.println("weather8.txt:");  
  // read from the file until there's nothing else in it:
    while (myFile.available()) {
    	Serial.write(myFile.read());
    }
	// close the file:
    myFile.close();
    } 
    else {
        // if the file didn't open, print an error:
        Serial.println("error opening test.txt");
      }
      lcd.clear();
    }
```
