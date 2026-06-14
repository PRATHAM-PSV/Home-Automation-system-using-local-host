# Offline Decentralized Home Automation System

A budget-friendly, internet-independent home automation system designed to control appliances across multiple physical locations without a local router or an active internet connection. 

The project leverages an **ESP32** acting as a local web server (Access Point mode) for smartphone control, coupled with **NRF24L01** radio modules to extend control to a secondary location without duplicating expensive microcontroller hardware.

---

## How It Works

* **Primary Hub (Location 1):** An ESP32 hosts a local website. Your smartphone connects directly to the ESP32's built-in Wi-Fi network (AP Mode). The ESP32 handles commands for local relays and reads physical manual switches.
* **Radio Bridge:** When an appliance at the second location needs to be toggled, the ESP32 transmits a wireless command via an **NRF24L01 (2.4GHz)** transceiver module.
* **Secondary Node (Location 2):** An **Arduino Nano** paired with a second NRF24L01 module listens for these radio signals, instantly triggering the local relays at the second location.

---

## Key Features

* **100% Offline & Private:** Zero reliance on cloud servers, external internet, or home Wi-Fi routers.
* **Hybrid Control:** Seamlessly switch between smartphone web UI control and traditional manual wall switches.
* **Cost-Optimized Architecture:** Uses a single ESP32 for the web interface, utilizing a cheaper Arduino Nano + NRF pair to extend the range instead of buying a second Wi-Fi-capable chip.
* **Hardware Fail-Safe:** State synchronization ensures manual switches still work even if the wireless network is pairing.

---

## Hardware Components

* **Microcontrollers:** ESP32 (Master Hub), Arduino Nano (Slave Node)
* **Wireless Communication:** NRF24L01 2.4GHz Radio Modules
* **Actuators & Inputs:** 5V Relay Modules, Physical Toggle Switches
