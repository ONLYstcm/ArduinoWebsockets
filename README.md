[![arduino-library-badge](https://www.ardu-badge.com/badge/ArduinoWebsockets.svg?)](https://www.ardu-badge.com/ArduinoWebsockets)  [![Build Status](https://travis-ci.org/gilmaimon/ArduinoWebsockets.svg?branch=master)](https://travis-ci.org/gilmaimon/ArduinoWebsockets)

# Arduino Websockets

A library for writing modern websockets applications with Arduino (ESP8266 and ESP32). This project is based on my project [TinyWebsockets](https://github.com/gilmaimon/TinyWebsockets).

The library provides simple and easy interface for websockets work (Client and Server). See the [basic-usage](#Basic-Usage) guide and the [examples](#Full-Examples).

## Getting Started
This section should help you get started with the library. If you have any questions feel free to open an issue.

### Prerequisites
Currently (version 0.4.0) the library only works with `ESP8266` and `ESP32`.

### Installing

You can install the library from the Arduino IDE or using a release ZIP file from the [Github realese page](https://github.com/gilmaimon/ArduinoWebsockets/releases).
Detailed instructions can be found [here](https://www.ardu-badge.com/ArduinoWebsockets).

## Basic Usage

### Client

Creating a client and connecting to a server:
```c++
WebsocketsClient client;
client.connect("ws://your-server-ip:port/uri");
```

Sending a message:
```c++
client.send("Hello Server!");
```

Waiting for messages:
```c++
client.onMessage([](WebsocketsMessage msg){
    Serial.prinln("Got Message: " + msg.data());
});
```

In order to keep receiving messages, you should:
```c++
void loop() {
    client.poll();
}
```
### Server

Creating a server and listening for connections:
```c++
WebsocketsServer server;
server.listen(8080);
```

Accepting connections:
```c++
WebsocketsClient client = server.accept();
// handle client as described before :)
```

## Full Examples

### Client

```c++
#include <ArduinoWebsockets.h>
#include <ESP8266WiFi.h>

const char* ssid = "ssid"; //Enter SSID
const char* password = "password"; //Enter Password
const char* websockets_server = "www.myserver.com:8080"; //server adress and port

using namespace websockets;

void onMessageCallback(WebsocketsMessage message) {
    Serial.print("Got Message: ");
    Serial.println(message.data());
}

void onEventsCallback(WebsocketsEvent event, String data) {
    if(event == WebsocketsEvent::ConnectionOpened) {
        Serial.println("Connnection Opened");
    } else if(event == WebsocketsEvent::ConnectionClosed) {
        Serial.println("Connnection Closed");
    } else if(event == WebsocketsEvent::GotPing) {
        Serial.println("Got a Ping!");
    } else if(event == WebsocketsEvent::GotPong) {
        Serial.println("Got a Pong!");
    }
}

WebsocketsClient client;
void setup() {
    Serial.begin(115200);
    // Connect to wifi
    WiFi.begin(ssid, password);

    // Wait some time to connect to wifi
    for(int i = 0; i < 10 && WiFi.status() != WL_CONNECTED; i++) {
        Serial.print(".");
        delay(1000);
    }

    // Setup Callbacks
    client.onMessage(onMessageCallback);
    client.onEvent(onEventsCallback);
    
    // Connect to server
    client.connect(websockets_server);

    // Send a message
    client.send("Hi Server!");
    // Send a ping
    client.ping();
}

void loop() {
    client.poll();
}
```
***Note:** for ESP32 you only need to change to code that connects to WiFi (replace `#include <ESP8266WiFi.h>` with `#include <WiFi.h>`), everything else stays the same.*

### Server
```c++
#include <ArduinoWebsockets.h>
#include <ESP8266WiFi.h>

const char* ssid = "ssid"; //Enter SSID
const char* password = "password"; //Enter Password

using namespace websockets;

WebsocketsServer server;
void setup() {
  Serial.begin(115200);
  // Connect to wifi
  WiFi.begin(ssid, password);

  // Wait some time to connect to wifi
  for(int i = 0; i < 15 && WiFi.status() != WL_CONNECTED; i++) {
      Serial.print(".");
      delay(1000);
  }
  
  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());   //You can get IP address assigned to ESP

  server.listen(80);
  Serial.print("Is server live? ");
  Serial.println(server.available());
}

void loop() {
  auto client = server.accept();
  if(client.available()) {
    auto msg = client.readBlocking();

    // log
    Serial.print("Got Message: ");
    Serial.println(msg.data());

    // return echo
    client.send("Echo: " + msg.data());

    // close the connection
    client.close();
  }
  
  delay(1000);
}
```
***Note:** for ESP32 you only need to change to code that connects to WiFi (replace `#include <ESP8266WiFi.h>` with `#include <WiFi.h>`), everything else stays the same.*

## SSL and WSS Support

No matter what board you are using, in order to use WSS (websockets over SSL) you need to use
```c++
client.connect("wss://your-secured-server-ip:port/uri");
```

The next sections describe board-specific code for using WSS with the library.

### ESP8266
With the esp8266 there are 2 ways for using WSS. By default, `ArduinoWebsockets` does not validate the certificate chain. This can be set explicitly using:
```c++
client.setInsecure();
```

You can also use a `SSL Fingerprint` to validate the SSL connection, for example:
```c++
const char ssl_fingerprint[] PROGMEM = "D5 07 4D 79 B2 D2 53 D7 74 E6 1B 46 C5 86 4E FE AD 00 F1 98";

client.setFingerprint(ssl_fingerprint);
```

### ESP32
With the esp32 there is only one option, `ArduinoWebsockets` supports validating the certificate chain by providing the `SSL Certificate` for the server's CA, for example:
```c++
const char ssl_ca_cert[] PROGMEM = \
    "-----BEGIN CERTIFICATE-----\n" \
    "MIIEkjCCA3qgAwIBAgIQCgFBQgAAAVOFc2oLheynCDANBgkqhkiG9w0BAQsFADA/\n" \
    "MSQwIgYDVQQKExtEaWdpdGFsIFNpZ25hdHVyZSBUcnVzdCBDby4xFzAVBgNVBAMT\n" \
    "DkRTVCBSb290IENBIFgzMB4XDTE2MDMxNzE2NDA0NloXDTIxMDMxNzE2NDA0Nlow\n" \
    "SjELMAkGA1UEBhMCVVMxFjAUBgNVBAoTDUxldCdzIEVuY3J5cHQxIzAhBgNVBAMT\n" \
    "GkxldCdzIEVuY3J5cHQgQXV0aG9yaXR5IFgzMIIBIjANBgkqhkiG9w0BAQEFAAOC\n" \
    "AQ8AMIIBCgKCAQEAnNMM8FrlLke3cl03g7NoYzDq1zUmGSXhvb418XCSL7e4S0EF\n" \
    "q6meNQhY7LEqxGiHC6PjdeTm86dicbp5gWAf15Gan/PQeGdxyGkOlZHP/uaZ6WA8\n" \
    "SMx+yk13EiSdRxta67nsHjcAHJyse6cF6s5K671B5TaYucv9bTyWaN8jKkKQDIZ0\n" \
    "Z8h/pZq4UmEUEz9l6YKHy9v6Dlb2honzhT+Xhq+w3Brvaw2VFn3EK6BlspkENnWA\n" \
    "a6xK8xuQSXgvopZPKiAlKQTGdMDQMc2PMTiVFrqoM7hD8bEfwzB/onkxEz0tNvjj\n" \
    "/PIzark5McWvxI0NHWQWM6r6hCm21AvA2H3DkwIDAQABo4IBfTCCAXkwEgYDVR0T\n" \
    "AQH/BAgwBgEB/wIBADAOBgNVHQ8BAf8EBAMCAYYwfwYIKwYBBQUHAQEEczBxMDIG\n" \
    "CCsGAQUFBzABhiZodHRwOi8vaXNyZy50cnVzdGlkLm9jc3AuaWRlbnRydXN0LmNv\n" \
    "bTA7BggrBgEFBQcwAoYvaHR0cDovL2FwcHMuaWRlbnRydXN0LmNvbS9yb290cy9k\n" \
    "c3Ryb290Y2F4My5wN2MwHwYDVR0jBBgwFoAUxKexpHsscfrb4UuQdf/EFWCFiRAw\n" \
    "VAYDVR0gBE0wSzAIBgZngQwBAgEwPwYLKwYBBAGC3xMBAQEwMDAuBggrBgEFBQcC\n" \
    "ARYiaHR0cDovL2Nwcy5yb290LXgxLmxldHNlbmNyeXB0Lm9yZzA8BgNVHR8ENTAz\n" \
    "MDGgL6AthitodHRwOi8vY3JsLmlkZW50cnVzdC5jb20vRFNUUk9PVENBWDNDUkwu\n" \
    "Y3JsMB0GA1UdDgQWBBSoSmpjBH3duubRObemRWXv86jsoTANBgkqhkiG9w0BAQsF\n" \
    "AAOCAQEA3TPXEfNjWDjdGBX7CVW+dla5cEilaUcne8IkCJLxWh9KEik3JHRRHGJo\n" \
    "uM2VcGfl96S8TihRzZvoroed6ti6WqEBmtzw3Wodatg+VyOeph4EYpr/1wXKtx8/\n" \
    "wApIvJSwtmVi4MFU5aMqrSDE6ea73Mj2tcMyo5jMd6jmeWUHK8so/joWUoHOUgwu\n" \
    "X4Po1QYz+3dszkDqMp4fklxBwXRsW10KXzPMTZ+sOPAveyxindmjkW8lGy+QsRlG\n" \
    "PfZ+G6Z6h7mjem0Y+iWlkYcV4PIWL1iwBi8saCbGS5jN2p8M+X+Q7UNKEkROb3N6\n" \
    "KOqkqm57TH2H3eDJAkSnh6/DNFu0Qg==\n" \
    "-----END CERTIFICATE-----\n";

client.setCACert(ssl_ca_cert);
```

## Contributing
Contributions are welcomed! Please open issues if you have troubles while using the library or any queshtions on how to get started. Pull requests are welcomed, please open an issue first.

## Change Log
- **14/02/2019 (v0.1.1)** - Initial commits and support for ESP32 and ESP8266 Websocket Clients.
- **16/02/2019 (v0.1.2)** - Added support for events (Pings, Pongs) and more internal improvements (events handling according to [RFC-6455](https://tools.ietf.org/html/rfc6455))
- **20/02/2019 (v0.1.3)** - Users now dont have to specify TCP client types (ESP8266/ESP32) they are selected automatically.
- **21/02/2019 (v0.1.5)** - Bug Fixes. Client now exposes a single string connect interface.
- **24/02/2019 (v0.2.0)** - User-facing interface is now done with Arduino's `String` class. Merged more changes (mainly optimizations) from TinyWebsockets.
- **25/02/2019 (v0.2.1)** - A tiny patch. Fixed missing user-facing strings for client interface. 
- **07/03/2019 (v0.3.0)** - A version update. Now supports a websockets server, better support for fragmented messages and streams. bug fixes and more optimized networking implementations. 
- **08/03/2019 (v0.3.1)** - Small patch. Merged changes from TinyWebsockets - interface changes to callbacks (partial callbacks without WebsocketsClient& as first parameter).
- **12/03/2019 (v0.3.2)** - Fixed a bug with behaviour of WebsokcetsClient (copy c'tor and assignment operator). Added close codes from TinyWebsockets. Thank you [@ramdor](https://github.com/gilmaimon/ArduinoWebsockets/issues/2)
- **13/03/2019 (v0.3.3)** - Fixed a bug in the esp8266 networking impl. Thank you [@ramdor](https://github.com/gilmaimon/ArduinoWebsockets/issues/2)
- **14/03/2019 (v0.3.4)** - changed underling tcp impl for esp8266 and esp32 to use `setNoDelay(true)` instead of sync communication. This makes communication faster and more relaiable than default. Thank you @ramdor for pointing out these methods.
- **06/04/2019 (v0.3.5)** - added very basic support for WSS in esp8266 (no support for fingerprint/ca or any kind of chain validation). 
- **22/04/2019 (v0.4.0)** - Added WSS support for both esp8266 and esp32. E328266 can use `client.setInsecure()` (does not validate certificate chain) or `client.setFingerprint(fingerprint)` in order to use WSS. With ESP32 there is `client.setCACert(certificate)` that can be used. (Usage is same as the built in `WiFiClientSecure`).  