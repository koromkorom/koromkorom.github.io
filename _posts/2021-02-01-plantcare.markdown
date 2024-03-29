---
layout: post
title:  "Plantcare!"
date:   2021-02-01 14:35:38 +0100
categories: iot
---
This is an IoT project using the google cloud platform. I developed an ESP32 based hardware device. The Firmware is updated over the air after CI/CD in Google Cloud Build. I present here the main cpp file of the firmware and the cloud bild file. 

Let's chechout the code:

```cpp
#include <Arduino.h>
#include <CloudIoTCore.h>
#include <CloudIoTCoreMqtt.h>
#include <WiFi.h>
#include <MQTT.h>
#include "ciotc_config.h" // Configure with your settings
#include <time.h>
#include <WiFiClientSecure.h>
#include <driver/adc.h>
#include <esp_wifi.h>
#include <Update.h>
```

Most of these libraries are pretty comon. 
The Cloud IoT Core library comes from here: <https://github.com/GoogleCloudPlatform/google-cloud-iot-arduino>
It has tools to connect IoT devices to the Google Cloud IoT Core. 

Some of the other libraries come from here: <https://github.com/espressif/arduino-esp32> 
This repository is maintained by espressif, the vendor of the ESP32 chip. 

```cpp
#define uS_TO_S_FACTOR 1000000ull /* Conversion factor for micro seconds to seconds */
#define TIME_TO_SLEEP 7200     /* Time ESP32 will go to sleep (in seconds) */

// These variables are stored in the RTC Memory and thereby survive reboot. 
RTC_DATA_ATTR int bootCount = 0;
RTC_DATA_ATTR int32_t uptime = 0;

```
The RTC_DATA_ATTR macros are special in the ESP32, because they survive reboot. 
So I use them here to count boots.

```cpp

#if ARDUHAL_LOG_LEVEL >= ARDUHAL_LOG_LEVEL_DEBUG
#include "esp32-hal-log.h"
#define DEBUG_MSG(...) log_d("%s",String( __VA_ARGS__ ).c_str() )
#define DEBUG_ENABLE true
#define DEBUG_UPTIME String((int32_t)esp_timer_get_time() / 1000)
#else
#define DEBUG_MSG(...)
#define DEBUG_ENABLE false
#endif

```
Here I define some debugging macros. It makes sense to turn this off in production. 
Too much serial output draws a lot of power from a battery powered device. 

```cpp

CloudIoTCoreDevice *device;
CloudIoTCoreMqtt *mqtt;
WiFiClientSecure *netClient;
MQTTClient *mqttClient;

unsigned long iss = 0;
String jwt;

esp_sleep_wakeup_cause_t wakeup_reason;

long lastMsg = 0;
long lastDisconnect = 0;
char msg[20];
int counter = 0;
int forceJwtExpSecs = 3600; // One hour is typical, up to 24 hours.

uint16_t sensfs;
uint16_t sensfm;
uint16_t sensfl;
uint16_t sensb;
uint16_t volt;
int8_t power = 0;

```
These are just some declarations and initializiations.
Now lets have a look at the OTA update.
```cpp
// Utility to extract header value from headers
String getHeaderValue(String header, String headerName) {
  return header.substring(strlen(headerName.c_str()));
}

// OTA Logic 
void execOTA(String newVersion) {
  WiFiClientSecure client;
  int contentLength = 0;
  bool isValidContentType = false;

  // Google Storage Bucket Config
  String host = "storage.googleapis.com"; // Host => storage.googleapis.com
  int port = 443; // Non https. For HTTPS 443. As of today, HTTPS doesn't work.
  String bin = "/plantcare-firmware/v" + String(newVersion)  + "/firmware.bin"; // bin file name with a slash in front.
  client.setCACert(ota_root_ca);
  Serial.println("Connecting to: " + String(host));
  // Connect to Google Storage
  if (client.connect(host.c_str(), port)) {
    // Connection Succeed.
    // Fecthing the bin
    Serial.println("Fetching Bin: " + String(bin));

    // Get the contents of the bin file
    client.print(String("GET ") + bin + " HTTP/1.1\r\n" +
                 "Host: " + host + "\r\n" +
                 "Cache-Control: no-cache\r\n" +
                 "Connection: close\r\n\r\n");

    unsigned long timeout = millis();
    while (client.available() == 0) {
      if (millis() - timeout > 10000) {
        Serial.println("Client Timeout !");
        client.stop();
        return;
      }
    }
    // Once the response is available,
    // check stuff

    while (client.available()) {
      // read line till /n
      String line = client.readStringUntil('\n');
            
      // remove space, to check if the line is end of headers
      line.trim();

      // if the the line is empty,
      // this is end of headers
      // break the while and feed the
      // remaining `client` to the
      // Update.writeStream();
      if (!line.length()) {
        //headers ended
        break; // and get the OTA started
      }

      // Check if the HTTP Response is 200
      // else break and Exit Update
      if (line.startsWith("HTTP/1.1")) {
        if (line.indexOf("200") < 0) {
          Serial.println("Got a non 200 status code from server. Exiting OTA Update.");
          break;
        }
      }

      // extract headers here
      // Start with content length
      if (line.startsWith("Content-Length: ")) {
        contentLength = atoi((getHeaderValue(line, "Content-Length: ")).c_str());
        Serial.println("Got " + String(contentLength) + " bytes from server");
      }

      // Next, the content type
      if (line.startsWith("Content-Type: ")) {
        String contentType = getHeaderValue(line, "Content-Type: ");
        Serial.println("Got " + contentType + " payload.");
        if (contentType == "application/octet-stream") {
          isValidContentType = true;
        }
      }
    }
  } else {
    // Connect to Google failed
    // Could be network problems 
    Serial.println("Connection to " + String(host) + " failed. Please check your setup");
    // retry?
    // execOTA();
  }

  // Check what is the contentLength and if content type is `application/octet-stream`
  Serial.println("contentLength : " + String(contentLength) + ", isValidContentType : " + String(isValidContentType));

  // check contentLength and content type
  if (contentLength && isValidContentType) {
    // Check if there is enough to OTA Update
    bool canBegin = Update.begin(contentLength);

    // If yes, begin
    if (canBegin) {
      Serial.println("Begin OTA. This may take 2 - 5 mins to complete. Things might be quite for a while.. Patience!");
      // No activity would appear on the Serial monitor
      // So be patient. This may take 2 - 5mins to complete
      size_t written = Update.writeStream(client);

      if (written == contentLength) {
        Serial.println("Written : " + String(written) + " successfully");
      } else {
        Serial.println("Written only : " + String(written) + "/" + String(contentLength) + ". Retry?" );
        // retry??
        // execOTA();
      }

      if (Update.end()) {
        Serial.println("OTA done!");
        if (Update.isFinished()) {
          Serial.println("Update successfully completed. Rebooting.");
          ESP.restart();
        } else {
          Serial.println("Update not finished? Something went wrong!");
        }
      } else {
        Serial.println("Error Occurred. Error #: " + String(Update.getError()));
      }
    } else {
      // not enough space to begin OTA
      // Understand the partitions and
      // space availability
      Serial.println("Not enough space to begin OTA");
      client.flush();
    }
  } else {
    Serial.println("There was no content in the response");
    client.flush();
  }
}
```
This function simply calls an HTTPS GET request to get the content and length of the new firmware binary to download. 
If checks are ok, the download is executed. 

```cpp
int versionCompare(String v1, String v2) 
{ 
    //  vnum stores each numeric part of version 
    int vnum1 = 0, vnum2 = 0; 
  
    //  loop untill both string are processed 
    for (int i=0,j=0; (i<v1.length() || j<v2.length()); ) 
    { 
        //  storing numeric part of version 1 in vnum1 
        while (i < v1.length() && v1[i] != '.') 
        { 
            vnum1 = vnum1 * 10 + (v1[i] - '0'); 
            i++; 
        } 
  
        //  storing numeric part of version 2 in vnum2 
        while (j < v2.length() && v2[j] != '.') 
        { 
            vnum2 = vnum2 * 10 + (v2[j] - '0'); 
            j++; 
        } 
  
        if (vnum1 > vnum2) 
            return 1; 
        if (vnum2 > vnum1) 
            return -1; 
  
        //  if equal, reset variables and go for next numeric 
        // part 
        vnum1 = vnum2 = 0; 
        i++; 
        j++; 
    } 
    return 0; 
} 

// This handles incoming mqtt messages
void messageReceived(String &topic, String &payload) {
  Serial.println("incoming: " + topic + " - " + payload);
  String onlineVersion = payload; 
  // Compare versions
  if (versionCompare(VERSION, onlineVersion) < 0) {
      DEBUG_MSG("New Version available. Starting Update.");
      execOTA(onlineVersion);
  }
  else if (versionCompare(VERSION, onlineVersion) > 0) 
      DEBUG_MSG("This version is newer than online version.");
  else
      DEBUG_MSG("Firmware up to date!");
}

String getJwt() {
  iss = time(nullptr);
  DEBUG_MSG("Refreshing JWT");
  jwt = device->createJWT(iss, jwt_exp_secs);
  return jwt;
}

```
Here we have a few helper functions. 
Nothing special, but it surprised me, that comparing version numbers
is such a hustle.

```cpp

bool initClients()
{
  device = new CloudIoTCoreDevice(project_id, location, registry_id,
                                  device_id, private_key_str);
  mqttClient = new MQTTClient(512);
  mqttClient->setOptions(180, true, 1000); // keepAlive, cleanSession, timeout
  netClient = new WiFiClientSecure();
  netClient->setCACert (root_cert_lts);
  mqtt = new CloudIoTCoreMqtt(mqttClient, netClient, device);
  mqtt->setUseLts(true);
  mqtt->startMQTT();

  DEBUG_MSG("Connecting to " + String(CLOUD_IOT_CORE_MQTT_HOST_LTS));
  
  // Establish LTS Connection
  int ret = netClient->connect(CLOUD_IOT_CORE_MQTT_HOST_LTS, (uint16_t) CLOUD_IOT_CORE_MQTT_PORT);
  if(ret == 0 && netClient->lastError(nullptr, 0) == -9984){
    DEBUG_MSG("Problem with Root CA. Trying backup.");
    netClient->setCACert(root_cert_lts_backup);
  }
  ret = netClient->connect(CLOUD_IOT_CORE_MQTT_HOST_LTS, (uint16_t) CLOUD_IOT_CORE_MQTT_PORT);
  if(ret == 0 && netClient->lastError(nullptr, 0) == -9984){
    DEBUG_MSG("Again problem with Root CA. Cancel.");
    return false;
  }
  if (ret == 0) {
    DEBUG_MSG("Problem with WIFIClientSecure. Error Code: " + String(ret));
    return false;
  }

  mqtt->mqttConnect(true);

  lastDisconnect = millis();

  return true;
}

bool reinitClients()
{
  delete (mqttClient);
  delete (netClient);
  delete (device);
  delete (mqtt);
  bool res = initClients();
  return res;
}

```
Here I initialize a client for the MQTT protocol and a client for
networking. The issue here is security. LTS is used so we need a 
root CA.
```cpp
/*
Method to sync time with ntp server
*/
time_t sync_time(){
    // Timesync
    DEBUG_MSG("Timesync started. uptime_Timesync_init:" + DEBUG_UPTIME);
    configTime(0, 0, "time.google.com", "pool.ntp.org", "time.nist.gov");
    DEBUG_MSG("Waiting on time sync...");
    while (time(nullptr) < 1510644967)
    {
      delay(10);
    }
    time_t now = time(nullptr);
    DEBUG_MSG("Timesync done. Time: " + String(ctime(&now)) + " uptime_Timesync_done:" + DEBUG_UPTIME);
    return now;
}

```
This seems small, but is a big issue. Acurate time is important for 
security protocolls and the sync itself took pretty long and I couldn't 
figure out why exactly that is. My guess is, that time sync protocolls, 
work with convergence somehow. 
```cpp

/*
Method to print the reason by which ESP32
has been awaken from sleep
*/
void print_wakeup_reason()
{
  wakeup_reason = esp_sleep_get_wakeup_cause();

  switch (wakeup_reason)
  {
  case ESP_SLEEP_WAKEUP_UNDEFINED:
    DEBUG_MSG("Reset was not caused by waking up from deep sleep.");
    break;
  case ESP_SLEEP_WAKEUP_ALL:
    DEBUG_MSG("Not a wakeup cause, used to disable all wakeup sources with esp_sleep_disable_wakeup_source. ");
    break;
  case ESP_SLEEP_WAKEUP_EXT0:
    DEBUG_MSG("Wakeup caused by external signal using RTC_IO");
    break;
  case ESP_SLEEP_WAKEUP_EXT1:
    DEBUG_MSG("Wakeup caused by external signal using RTC_CNTL");
    break;
  case ESP_SLEEP_WAKEUP_TIMER:
    DEBUG_MSG("Wakeup caused by timer");
    break;
  case ESP_SLEEP_WAKEUP_TOUCHPAD:
    DEBUG_MSG("Wakeup caused by touchpad");
    break;
  case ESP_SLEEP_WAKEUP_ULP:
    DEBUG_MSG("Wakeup caused by ULP program");
    break;
  default:
    DEBUG_MSG("Wakeup cause unknown");
    break;
  }
}

void printLocalTime()
{
  struct tm timeinfo;
  if(!getLocalTime(&timeinfo)){
    log_d("Failed to obtain time");
    return;
  }
  log_d("bootuptime_readable: ");
  if(DEBUG_ENABLE) Serial.println(&timeinfo, "%A, %B %d %Y %H:%M:%S");
}

void dns(){

  IPAddress ServerIP;
  const char mqttServerName[] = CLOUD_IOT_CORE_MQTT_HOST_LTS;
  DEBUG_MSG("Trying to solve Hostname " + String(mqttServerName));     
  WiFi.hostByName(mqttServerName, ServerIP);
  DEBUG_MSG("IP: " + ServerIP.toString());
}

```
Here we have again a few helper functions. I want to mention the dns() function, which simply tries to resolve the hostname of the mqtt server. 
This also took quite long on the ESP, which surprised me a little. Now we are where the fun begins. setup() is the main function of any android based code. I will split up this funtion here because there is a lot going on.
```cpp
void setup()
{

  // put your setup code here, to run once:
  if(DEBUG_ENABLE) Serial.begin(115200);

  //Increment boot number and print it every reboot
  time_t bootuptime;
  time(&bootuptime);
  DEBUG_MSG("Initialization done. Boot number: " + String(bootCount) +" bootupime: "+bootuptime+ " uptime0:" + DEBUG_UPTIME);
  bootCount++;
  DEBUG_MSG("Firmware Version: " + String(VERSION));
  if(DEBUG_ENABLE) printLocalTime();

  //Print the wakeup reason for ESP32
  print_wakeup_reason();
  
```
Just a few wake up messages. This is important, because we want to know,
why it woke up. 
```cpp
  // Setup Sensor channel
   touch_pad_init();

  // Setup Voltage channel
  adc1_config_width(ADC_WIDTH_BIT_12);
  adc1_config_channel_atten(ADC1_CHANNEL_3, ADC_ATTEN_DB_11);

  // WiFi
  DEBUG_MSG("Connecting to WiFi ... Status:" + String(WiFi.status()) + " uptime_WiFi_init:" + DEBUG_UPTIME);
  DEBUG_MSG("SSID:" + String(ssid) + " pw:" + String(password));
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  int cnt = 0;
  while (WiFi.status() != WL_CONNECTED)
  {
        delay(500);
    Serial.print(".");
    if (cnt++ >= 10)
    {
      WiFi.beginSmartConfig();
      while (1)
      {
        delay(1000);
        if (WiFi.smartConfigDone())
        {
          Serial.println("SmartConfig Success");
          strncpy( ssid, WiFi.SSID().c_str(), WiFi.SSID().length() );
          ssid[WiFi.SSID().length()] = 0;
          strncpy( password, WiFi.psk().c_str(), WiFi.psk().length() );
          password[WiFi.psk().length() ] = 0;
          Serial.println(ssid);
          Serial.println(password);
          break;
        }
        else
        {
          Serial.println("connecting...");
        }
      }
    }
  }
  DEBUG_MSG("Connected to WiFi. uptime_WiFi_done:" + DEBUG_UPTIME);
```
Here the wifi connection is established. This worked really well. 

```cpp
  // DNS 
  DEBUG_MSG("Resolving hostname. uptime_DNS_init:" + DEBUG_UPTIME);
  dns();
  DEBUG_MSG("Got IP Adress. uptime_DNS_done:" + DEBUG_UPTIME);

```
I decided to seperate DNS, because it took a long time during booting. 
```cpp

  if(wakeup_reason == ESP_SLEEP_WAKEUP_UNDEFINED)
  {
    DEBUG_MSG("Boot up caused by reset. No Deep Sleep. Syncing time.");
    sync_time();
  }

```
Here I checked wether boot up was caused by pushing a button on the device itself.
```cpp

  // Init and Connect MQTT
  DEBUG_MSG("Init MQTT started. uptime_MQTT_init_init:" + DEBUG_UPTIME);
  bool res = initClients();
  // Only do this, when initialization is successful.   
  if (res) {
    DEBUG_MSG("Init MQTT started. uptime_MQTT_init_done:" + DEBUG_UPTIME);
    DEBUG_MSG("Init MQTT started. uptime_MQTT_init:" + DEBUG_UPTIME);
    int loopReturn = mqttClient->loop();
    lwmqtt_return_code_t returnCode = mqttClient->returnCode();
    int reconnect_count = 0;
    if(returnCode == LWMQTT_NOT_AUTHORIZED)
    {
      DEBUG_MSG("MQTT Bad Client ID. Maybe time out of sync. Syncing time.");
      time_t ntp = sync_time();
      DEBUG_MSG("Time was off by [bootup-ntp]:" + String(bootuptime-ntp));
    }
    if(returnCode == LWMQTT_BAD_USERNAME_OR_PASSWORD)
    {
      DEBUG_MSG("MQTT Bad Credentials. Maybe time out of sync. Syncing time.");
      time_t ntp = sync_time();
      DEBUG_MSG("Time was off by [bootup-ntp]:" + String(bootuptime-ntp));
    }
```
Many problems were caused by an off system clock. It is a good thing, this is 
failing. Otherwise this could be exploited. 
```cpp
    while (returnCode != LWMQTT_CONNECTION_ACCEPTED)
    {
      DEBUG_MSG("Reinitiating clients. Count: " + String(reconnect_count));
      reinitClients();
      loopReturn = mqttClient->loop();
      reconnect_count++;
    }
    DEBUG_MSG("Init and Connect MQTT done. uptime_MQTT_done:" + DEBUG_UPTIME);
```
Sometimes it helps to just reinitialize everything. 
```cpp
    // Read sensor data and pack msg
    DEBUG_MSG("Read Sensor data started. uptime_sens_init:" + DEBUG_UPTIME);
    time_t nownow;
    time(&nownow);
    sensfs = touchRead(T9);
    sensfm = touchRead(T2);
    sensfl = touchRead(T4);
    sensb = touchRead(T8);
    volt = adc1_get_raw(ADC1_CHANNEL_3);
    esp_wifi_get_max_tx_power(&power);

    String payload = String("{\"moistfs\":") + sensfs +
                    String(",\"moistfm\":") + sensfm +
                    String(",\"moistfl\":") + sensfl +
                    String(",\"moistb\":") + sensb +
                    String(",\"voltage\":") + volt +
                    String(",\"wifi_power\":") + power +
                    String(",\"uptime\":") + uptime +
                    String(",\"timestamp\":") + nownow +
                    String(",\"wakeup_reason\":") + (int)wakeup_reason +
                    String(",\"bootCount\":") + bootCount +
                    String("}");
    DEBUG_MSG("Read Sensor data done uptime_sens_done:" + DEBUG_UPTIME);
```
The payload is just a long string, with all the sonsor data packed in it. 
```cpp
    // Sending MQTT message
    DEBUG_MSG("Sending MQTT message started. uptime_send_init:" + DEBUG_UPTIME);
    DEBUG_MSG(String("Sending String:") + payload);
    mqtt->publishTelemetry(payload);
    //DEBUG_MSG("Message Pubished. MQTT Client State: " + client->printmqttstate(client->getMqttState()) + " uptime_send_done:" + DEBUG_UPTIME);
```
Here the data is send to the google cloud mqtt interface. 
```cpp
    // Disconecting
    DEBUG_MSG("Disconnecting started. uptime_disc_init:" + DEBUG_UPTIME);
    mqttClient->disconnect();

  } else {
    DEBUG_MSG("Init Failed. Going to sleep again.");
  }
  /*
  First we configure the wake up source
  We set our ESP32 to wake up every 5 seconds
  */
  esp_sleep_enable_timer_wakeup(TIME_TO_SLEEP * uS_TO_S_FACTOR);
  DEBUG_MSG("Setup ESP32 to sleep for " + String(TIME_TO_SLEEP) +
            " Seconds");

  /*
  Next we decide what all peripherals to shut down/keep on
  By default, ESP32 will automatically power down the peripherals
  not needed by the wakeup source, but if you want to be a poweruser
  this is for you. Read in detail at the API docs
  http://esp-idf.readthedocs.io/en/latest/api-reference/system/deep_sleep.html
  Left the line commented as an example of how to configure peripherals.
  The line below turns off all RTC peripherals in deep sleep.
  */
  //esp_deep_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_OFF);
  //DEBUG_MSG("Configured all RTC Peripherals to be powered down in sleep");

  /*
  Now that we have setup a wake cause and if needed setup the
  peripherals state in deep sleep, we can now start going to
  deep sleep.
  In the case that no wake up sources were provided but deep
  sleep was started, it will sleep forever unless hardware
  reset occurs.
  */

  DEBUG_MSG("Going to sleep now. uptime_sleep:" + DEBUG_UPTIME);
  DEBUG_MSG("\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n");
  Serial.flush();

  // For DEBUGGING:
  //delay(18000);

  uptime = (int32_t)esp_timer_get_time() / 1000;

  esp_deep_sleep_start();
  DEBUG_MSG("This will never be printed");
}
```
This initialises a device specific feature. It can go into deep sleep 
for some time and wake up based on different causes. 



This code is now compiled inside a docker container running in google cloud. 
A push to github triggers the following simple cloudbuild script, which 
copies the binary into a new directory for each version.

```yaml
steps:
- name: 'gcr.io/plantcare01-223216/quickstart-image:tag1'  
artifacts: 
  objects: 
    location: 'gs://plantcare-firmware/$TAG_NAME'
    paths: ['.pio/build/esp32dev/firmware.bin']
```