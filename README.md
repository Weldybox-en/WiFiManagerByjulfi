Please notice that the documentation bellow is an adaptation of the one created by [tapzu](https://github.com/tzapu/WiFiManager) (owner of Wifimanager).

# WiFiManager
ESP8266 WiFi Connection manager with fallback web configuration portal

The configuration portal is of the captive variety, so on various devices it will present the configuration dialogue as soon as you connect to the created access point.

First attempt at a library. Lots more changes and fixes to do. Contributions are welcome.

#### This works with the ESP8266 Arduino platform with a recent stable release(2.0.0 or newer) https://github.com/esp8266/Arduino

## Contents
 - [How it works](#how-it-works)
 - [Wishlist](#wishlist)
 - [Quick start](#quick-start)
   - [Installing](#installing)
   - [Using](#using)
 - [Documentation](#documentation)
   - [Access Point Password](#password-protect-the-configuration-access-point)
   - [Callbacks](#callbacks)
   - [Configuration Portal Timeout](#configuration-portal-timeout)
   - [On Demand Configuration](#on-demand-configuration-portal)
   - [Custom Parameters](#custom-parameters)
   - [Custom IP Configuration](#custom-ip-configuration)
   - [Filter Low Quality Networks](#filter-networks)
   - [Debug Output](#debug)
 - [Troubleshooting](#troubleshooting)


## How It Works
- when your ESP starts up, it sets it up in Station mode and tries to connect to a previously saved Access Point
- if this is unsuccessful (or no previous network saved) it moves the ESP into Access Point mode and spins up a DNS and WebServer (default ip 192.168.4.1)
- using any wifi enabled device with a browser (computer, phone, tablet) connect to the newly created Access Point
- because of the Captive Portal and the DNS server you will either get a 'Join to network' type of popup or get any domain you try to access redirected to the configuration portal
- choose one of the access points scanned, enter password, click save
- ESP will try to connect. If successful, it relinquishes control back to your app. If not, reconnect to AP and reconfigure.

## How It Looks

<p>
  <img src="https://imgur.com/rtKI3Iw.png" width="379">
  <img src="https://imgur.com/GuqN9VI.png" width="350">
</p>

## Wishlist
- [x] remove dependency on EEPROM library
- [x] move HTML Strings to PROGMEM
- [x] cleanup and streamline code (although this is ongoing)
- [x] if timeout is set, extend it when a page is fetched in AP mode
- [x] add ability to configure more parameters than ssid/password
- [x] maybe allow setting ip of ESP after reboot
- [x] add to Arduino Library Manager
- [x] add to PlatformIO
- [ ] add multiple sets of network credentials
- [x] allow users to customize CSS
- [ ] ESP32 support or instructions
- [ ] rewrite documentation for simplicity, based on scenarios/goals
- [ ] rely on the SDK's built in auto connect more than forcing a connect

## Quick Start

### Installing

The installation is slightly different from the original. This one had to be installed manually!

- First, download the archive librairy
- Copy and past the directory in your IDE librairy managment directory. (Ardiuno IDE or Platformio)
- I personnaly use platformio and the correct directory is located here by default: *C:\Users\#username\.platformio\lib*

It look like this:

<p align="left">

  <img src="https://imgur.com/ycolq4p.png">
</p>

### Using
- Include in your sketch
```cpp
#include <ESP8266WiFi.h>          //ESP8266 Core WiFi Library (you most likely already have this in your sketch)

#include <DNSServer.h>            //Local DNS Server used for redirecting all requests to the configuration portal
#include <ESP8266WebServer.h>     //Local WebServer used to serve the configuration portal
#include <WiFiManagerByWeldy.h>          //https://github.com/tzapu/WiFiManager WiFi Configuration Magic
```

- Initialize library, in your setup function add
```cpp
WiFiManager wifiManager;
```

- Also in the setup function add
```cpp
//first parameter is name of access point, second is the password
wifiManager.autoConnect("AP-NAME", "AP-PASSWORD");
```
if you just want an unsecured access point
```cpp
wifiManager.autoConnect("AP-NAME");
```
or if you want to use and auto generated name from 'ESP' and the esp's Chip ID use
```cpp
wifiManager.autoConnect();
```

After you write your sketch and start the ESP, it will try to connect to WiFi. If it fails it starts in Access Point mode.
While in AP mode, connect to it then open a browser to the gateway IP, default 192.168.4.1, configure wifi, save and it should reboot and connect.

Also see [examples](https://github.com/tzapu/WiFiManager/tree/master/examples).

## Documentation

#### Password protect the configuration Access Point
You can and should password protect the configuration access point.  Simply add the password as a second parameter to `autoConnect`.
A short password seems to have unpredictable results so use one that's around 8 characters or more in length.
The guidelines are that a wifi password must consist of 8 to 63 ASCII-encoded characters in the range of 32 to 126 (decimal)
```cpp
wifiManager.autoConnect("AutoConnectAP", "password")
```

#### Callbacks
##### Enter Config mode
Use this if you need to do something when your device enters configuration mode on failed WiFi connection attempt.
Before `autoConnect()`
```cpp
wifiManager.setAPCallback(configModeCallback);
```
`configModeCallback` declaration and example
```cpp
void configModeCallback (WiFiManager *myWiFiManager) {
  Serial.println("Entered config mode");
  Serial.println(WiFi.softAPIP());

  Serial.println(myWiFiManager->getConfigPortalSSID());
}
```

##### Save settings
This gets called when custom parameters have been set **AND** a connection has been established. Use it to set a flag, so when all the configuration finishes, you can save the extra parameters somewhere.

See [AutoConnectWithFSParameters Example](https://github.com/tzapu/WiFiManager/tree/master/examples/AutoConnectWithFSParameters).
```cpp
wifiManager.setSaveConfigCallback(saveConfigCallback);
```
`saveConfigCallback` declaration and example
```cpp
//flag for saving data
bool shouldSaveConfig = false;

//callback notifying us of the need to save config
void saveConfigCallback () {
  Serial.println("Should save config");
  shouldSaveConfig = true;
}
```

#### Configuration Portal Timeout
If you need to set a timeout so the ESP doesn't hang waiting to be configured, for instance after a power failure, you can add
```cpp
wifiManager.setConfigPortalTimeout(180);
```
which will wait 3 minutes (180 seconds). When the time passes, the autoConnect function will return, no matter the outcome.
Check for connection and if it's still not established do whatever is needed (on some modules I restart them to retry, on others I enter deep sleep)

#### On Demand Configuration Portal
If you would rather start the configuration portal on demand rather than automatically on a failed connection attempt, then this is for you.

Instead of calling `autoConnect()` which does all the connecting and failover configuration portal setup for you, you need to use `startConfigPortal()`. __Do not use BOTH.__

Example usage
```cpp
void loop() {
  // is configuration portal requested?
  if ( digitalRead(TRIGGER_PIN) == LOW ) {
    WiFiManager wifiManager;
    wifiManager.startConfigPortal("OnDemandAP");
    Serial.println("connected...yeey :)");
  }
}
```
See example for a more complex version. [OnDemandConfigPortal](https://github.com/tzapu/WiFiManager/tree/master/examples/OnDemandConfigPortal)

#### Custom Parameters
You can use WiFiManager to collect more parameters than just SSID and password.
This could be helpful for configuring stuff like MQTT host and port, [blynk](http://www.blynk.cc) or [emoncms](http://emoncms.org) tokens, just to name a few.
**You are responsible for saving and loading these custom values.** The library just collects and displays the data for you as a convenience.
Usage scenario would be:
- load values from somewhere (EEPROM/FS) or generate some defaults
- add the custom parameters to WiFiManager using
 ```cpp
 // id/name, placeholder/prompt, default, length
 WiFiManagerParameter custom_mqtt_server("server", "mqtt server", mqtt_server, 40);
 wifiManager.addParameter(&custom_mqtt_server);

 ```
- if connection to AP fails, configuration portal starts and you can set /change the values (or use on demand configuration portal)
- once configuration is done and connection is established [save config callback]() is called
- once WiFiManager returns control to your application, read and save the new values using the `WiFiManagerParameter` object.
 ```cpp
 mqtt_server = custom_mqtt_server.getValue();
 ```  
This feature is a lot more involved than all the others, so here are some examples to fully show how it is done.
You should also take a look at adding custom HTML to your form.

- Save and load custom parameters to file system in json form [AutoConnectWithFSParameters](https://github.com/tzapu/WiFiManager/tree/master/examples/AutoConnectWithFSParameters)
- *Save and load custom parameters to EEPROM* (not done yet)

#### Custom IP Configuration
You can set a custom IP for both AP (access point, config mode) and STA (station mode, client mode, normal project state)

##### Custom Access Point IP Configuration
This will set your captive portal to a specific IP should you need/want such a feature. Add the following snippet before `autoConnect()`
```cpp
//set custom ip for portal
wifiManager.setAPStaticIPConfig(IPAddress(10,0,1,1), IPAddress(10,0,1,1), IPAddress(255,255,255,0));
```

##### Custom Station (client) Static IP Configuration
This will make use the specified IP configuration instead of using DHCP in station mode.
```cpp
wifiManager.setSTAStaticIPConfig(IPAddress(192,168,0,99), IPAddress(192,168,0,1), IPAddress(255,255,255,0));
```
There are a couple of examples in the examples folder that show you how to set a static IP and even how to configure it through the web configuration portal.

#### Custom HTML, CSS, Javascript
There are various ways in which you can inject custom HTML, CSS or Javascript into the configuration portal.
The options are:
- inject custom head element
You can use this to any html bit to the head of the configuration portal. If you add a `<style>` element, bare in mind it overwrites the included css, not replaces.
```cpp
wifiManager.setCustomHeadElement("<style>html{filter: invert(100%); -webkit-filter: invert(100%);}</style>");
```
- inject a custom bit of html in the configuration form
```cpp
WiFiManagerParameter custom_text("<p>This is just a text paragraph</p>");
wifiManager.addParameter(&custom_text);
```
- inject a custom bit of html in a configuration form element
Just add the bit you want added as the last parameter to the custom parameter constructor.
```cpp
WiFiManagerParameter custom_mqtt_server("server", "mqtt server", "iot.eclipse", 40, " readonly");
```

#### Filter Networks
You can filter networks based on signal quality and show/hide duplicate networks.

- If you would like to filter low signal quality networks you can tell WiFiManager to not show networks below an arbitrary quality %;
```cpp
wifiManager.setMinimumSignalQuality(10);
```
will not show networks under 10% signal quality. If you omit the parameter it defaults to 8%;

- You can also remove or show duplicate networks (default is remove).
Use this function to show (or hide) all networks.
```cpp
wifiManager.setRemoveDuplicateAPs(false);
```

#### Debug
Debug is enabled by default on Serial. To disable add before autoConnect
```cpp
wifiManager.setDebugOutput(false);
```

## Troubleshooting
If you get compilation errors, more often than not, you may need to install a newer version of the ESP8266 core for Arduino.

Changes added on 0.8 should make the latest trunk work without compilation errors. Tested down to ESP8266 core 2.0.0. **Please update to version 0.8**

I am trying to keep releases working with release versions of the core, so they can be installed through boards manager, but if you checkout the latest version directly from github, sometimes, the library will only work if you update the ESP8266 core to the latest version because I am using some newly added function.

If you connect to the created configuration Access Point but the configuration portal does not show up, just open a browser and type in the IP of the web portal, by default `192.168.4.1`.

If trying to connect ends up in an endless loop, try to add `setConnectTimeout(60)` before `autoConnect();`. The parameter is timeout to try connecting in seconds.

 ## Support me
 
I've decide to share for free my work because i've learn tanks to that! For me, sharing project like this for free accelerate the knowledge corculation and that very cool.

But i'm student and like every student i'm broken :(
So, if you can, feel free to buy me a coffe :D

[![paypal](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://www.buymeacoffee.com/jvZuQ8SYo)
