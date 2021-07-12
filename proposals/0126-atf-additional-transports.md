# ATF support of additional transports (BT and USB)

* Proposal: [SDL-0126](0126-atf-additional-transports.md)
* Author: [Alexander Kutsan](https://github.com/LuxoftAKutsan), [Oleksandr Deriabin](https://github.com/aderiabin)
* Status: **Awaiting review**
* Impacted Platforms: [ATF]

## Introduction

Currently some number of test scripts require manual checks using real transport. 
This part of manual work should be also automated with Automated test framework.

Automated test framework should support communication with SDL via :
 - TCP (already supported)
 - Bluetooth
 - USB
 
Full Bluetooth and USB testing of ATF should use real device for communication with SDL.
In future this approach also will allow to test SDL on custom OEM head units.

## Motivation

Some features of SDL assume usage of certain transport : USB or Bluetooth.
Some features describe SDL behavior in case of transport switch or multiple device connection.
Also generally SDL uses Bluetooth or USB as connection protocol on head unit. 
ATF should support custom transports. 

Main reasons :
 - Automated testing of transport specific use cases
 - Automated smoke and full regression testing of SDL via Bluetooth and USB
 - Automated testing of SDL on real hardware
 
## Proposed solution

Use real mobile device as a transport adapter.
Create Android mobile application as a part of ATF infrastructure.

Mobile application should use HTTP connection with REST protocol for control communication with ATF using next API:
 - GetDeviceInfo - Provides device's information
 - ConnectToSDL - Connects to SDL and provides proxy TCP connection parameters
 - DisconnectFromSDL - Disconnects from SDL
 - GetSDLConnectionStatus - Provides status of connection to SDL

Mobile application should provide next information as main part of the response to `GetDeviceInfo` request (all described parameters are mandatory):
 - Operating system type (string value from list: "Android", "iOS", "Linux", ...)
 - MAC address (string value)
 - Array of transports available to connect to SDL (string values from list: "WIFI", "BT", "USB")

Mobile application should provide host and port of TCP server for raw binary data communication with ATF on establishing successful connection to SDL.
Mentioned host and port should be provided as main part of the response to `ConnectToSDL` request.
Mobile application should transfer data from ATF received by TCP server to SDL and vice versa.
Mobile application may use [SDL android library](https://github.com/smartdevicelink/sdl_android) for communicating with SDL.

ATF should be able to connect to multiple Mobile transport adapters and provide ability to use any of them to test engineer.

Test engineers should be able to create sessions on provided by ATF mobile transport adapters.
Session interface should not be changed.

In the case where a mobile device is absent, ATF should be able to test SDL via TCP connection (as it does now).

High Level relationship diagram: 
![High Level relationship diagram](/assets/proposals/0126-ATF-Additional-Transports/atf_transport_adapter.png)

## Potential downsides

This solution is not scalable. 
To run multiple scripts that test transport simultaneously, it will require adding a physical mobile device.

## Impact on existing code

Impact on ATF internal structure.
May impact some scripts that test multiple connections.
Also if this solution uses sdl_android, some changes may be required in sdl_android. 

## Alternatives considered

#### Avoid automatic transport testing

 SDL has implemented transport using adapters, so porting SDL on customer hardware requires rewriting transport adapters from scratch.
 And any transport testing that is done on Ubuntu Linux x86 becomes not actual.
 But not having automated testing of transport makes it impossible to check the business logic that is related to the transport switch.
 In addition proposed approach provides possibility of SDL testing on custom transports.
 
 #### Emulate transports
 
 Emulating of USB/Bluetooth transports probably is possible, but it requires some research, development and also PoC project. 
 Also emulating of transport won't allow performing of automated testing of SDL on custom head units, so any testing that will be performed on Ubuntu Linux x86 becomes not actual. 
