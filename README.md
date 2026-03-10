
![Smart Clock](https://github.com/user-attachments/assets/6c34225a-b09b-4390-9ded-06280c461435)

# Raspberry Pi Smart Clock Dashboard

A lightweight **React-based smart dashboard** built for Raspberry Pi designed for continuous always-on display.
The project runs in **kiosk mode on Raspberry Pi OS** and displays real-time information such as the clock, weather, Spotify playback, and calendar events.

---

## Overview

This project aims to create a **low-power smart display optimized for embedded hardware** like the Raspberry Pi 3B+.

The interface is designed for a **1920×480 ultra-wide display** and runs in fullscreen kiosk mode using Chromium.

---

## Features

* Real-time clock display
* Weather data from external APIs
* Spotify “Now Playing” integration
* Google Calendar events panel
* Designed for always-on display
* Optimized for low-RAM hardware

---

## Tech Stack

* **React**
* **Vite**
* **JavaScript**
* **Framer Motion**
* **Raspberry Pi OS**
* **Chromium Kiosk Mode**
* **REST APIs**

---

## Hardware

Target device:

* Raspberry Pi 3B+
* Waveshare 8.8" DSI LCD (1920×480)
* Raspberry Pi OS Lite (64-bit)

---

## Project Structure

```
raspberry-pi-smart-clock
│
├── docs
│   └── technical-spec.md
│
├── src
│   ├── components
│   │   ├── Clock.jsx
│   │   ├── Weather.jsx
│   │   ├── NowPlaying.jsx
│   │   └── Calendar.jsx
│   │
│   ├── hooks
│   │   └── useTime.js
│   │
│   ├── services
│   │   └── weather.js
│   │
│   ├── App.jsx
│   └── main.jsx
│
├── config
│   ├── kiosk.service
│   └── openbox-autostart.sh
│
└── README.md
```

---

## Future Improvements

* Hardware monitoring (CPU temp, memory)
* Network monitoring panel
* Custom themes
* Touch input support
* Mobile remote control

---


**Zaki Syed**
Aspiring cybersecurity & systems engineer.
