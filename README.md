# Woman Safety — GPS SOS Tracker (Raspberry Pi Pico W + NEO-6M + R backend)
Project summary

A portable GPS SOS tracker built on a Raspberry Pi Pico W and NEO-6M GPS module. When a user presses the SOS button (or device detects an emergency), the device:

reads current GPS (lat, lon, time, speed),

sends an immediate alert (HTTP POST) to your server,

optionally sends SMS/WhatsApp/email alerts to preconfigured contacts,

provides live location tracking on a web dashboard built with R (plumber API + Shiny + leaflet),

logs events for later review.

This README includes hardware wiring, device firmware (MicroPython), a minimal R plumber endpoint to receive location pushes and send alerts, an R Shiny dashboard to show live location, database schema, security & privacy guidance, testing checklist, and deployment suggestions.

Table of Contents

Features

Hardware & Tools

Wiring (NEO-6M + Pico W + SOS Button)

Device firmware (MicroPython example)

Server: R plumber API (receive & log)

Dashboard: R Shiny app (live tracking using leaflet)

Database schema (SQLite)

Alerting (SMS / WhatsApp / email)

Security, privacy & legal considerations

Testing & debugging

Battery, enclosure & field notes

Known limitations & future improvements

1. Features

Real-time GPS location push (HTTP POST over HTTPS)

SOS button for immediate alert

Periodic heartbeat / location updates

Server-side logging (SQLite)

Live map dashboard built in R (leaflet)

Alert notifications via SMS (Twilio) or email

Simple authentication (API key) between device & server

2. Hardware & Tools

Hardware:

Raspberry Pi Pico W

NEO-6M GPS module (UART TX/RX, Vcc, GND)

Momentary push button (SOS)

10kΩ resistor (pull-down or pull-up depending wiring)

LiPo battery + step-up / power supply (or USB power bank)

Wires, breadboard, enclosure

Software / Services:

MicroPython firmware for Pico W

R (>= 4.0), packages: plumber, shiny, DBI, RSQLite, leaflet, httr, jsonlite

Optional: Twilio account (for SMS), email SMTP credentials, Firebase or push gateway

Host for plumber + Shiny (VPS, DigitalOcean droplet, nginx reverse proxy) or platforms that support R (shinyapps.io + external plumbing)

3. Wiring

Use the Pico W UART port (PIO UART or GP0/GP1 depending) — example pins below assume UART on pins GP0 (TX) and GP1 (RX) of Pico:

NEO-6M — Pico W

VCC -> 3.3V (or 5V if module requires; check your module specs)

GND -> GND

TX (GPS) -> Pico RX (GP1 / UART0 RX)

RX (GPS) -> Pico TX (GP0 / UART0 TX)

PPS (if present) -> optional GPIO (for high-precision timestamp)

SOS button:

One side -> Pico GPIO (e.g., GP15)

Other side -> GND

Use internal pull-up and detect press (active LOW) or wire to VCC and use pull-down; below firmware uses pull-up.

Important: confirm voltage levels. Many NEO-6M modules are 3.3V tolerant; some use 5V. Use level shifter if necessary.

4. Device firmware (MicroPython)

Save as main.py on the Pico W. This example:

reads NMEA sentences,

parses GGA for lat/lon (a simple parser),

debounces SOS button,

sends HTTP POST JSON to your server endpoint over Wi-Fi.

# main.py (MicroPython for Raspberry Pi Pico W)
import time
import network
import socket
import urequests as requests
import machine
from machine import UART, Pin

WIFI_SSID = "YOUR_SSID"
WIFI_PASS = "YOUR_PASS"
SERVER_URL = "https://yourserver.example.com/receive"  # plumber endpoint
API_KEY = "DEVICE_API_KEY"  # simple shared secret
SOS_PIN = 15
GPS_UART_ID = 0
GPS_BAUD = 9600
HEARTBEAT_SECONDS = 30

# Setup UART for GPS (UART0: TX GP0, RX GP1)
uart = UART(GPS_UART_ID, baudrate=GPS_BAUD, tx=machine.Pin(0), rx=machine.Pin(1))

# Setup SOS button (active low)
sos = Pin(SOS_PIN, Pin.IN, Pin.PULL_UP)

def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(WIFI_SSID, WIFI_PASS)
    for _ in range(30):
        if wlan.isconnected():
            print("Connected, IP:", wlan.ifconfig())
            return True
        time.sleep(1)
    print("WiFi connect failed")
    return False

def parse_nmea_latlon(nmea):
    # Very small NMEA parser for GGA or RMC
    # Returns (lat, lon, fix_time) or (None, None, None)
    try:
        parts = nmea.split(',')
        if parts[0].endswith('GGA'):
            lat_raw = parts[2]
            lat_dir = parts[3]
            lon_raw = parts[4]
            lon_dir = parts[5]
            time_utc = parts[1]
        elif parts[0].endswith('RMC'):
            lat_raw = parts[3]
            lat_dir = parts[4]
            lon_raw = parts[5]
            lon_dir = parts[6]
            time_utc = parts[1]
        else:
            return None, None, None
        if lat_raw == '' or lon_raw == '':
            return None, None, None
        # convert ddmm.mmmm to decimal degrees
        def conv(coord, direction):
            deg = float(coord[:2 if direction in ['N','S'] else 3])
            mins = float(coord[2 if direction in ['N','S'] else 3:])
            dec = deg + mins/60.0
            if direction in ['S','W']:
                dec = -dec
            return dec
        lat = conv(lat_raw, lat_dir)
        lon = conv(lon_raw, lon_dir)
        return lat, lon, time_utc
    except Exception as e:
        print("parse error", e)
        return None, None, None

def read_gps(timeout=2.0):
    start = time.time()
    line = b""
    while time.time() - start < timeout:
        if uart.any():
            c = uart.read(1)
            if not c:
                continue
            if c == b'\n':
                text = line.decode('utf-8', errors='ignore').strip()
                line = b""
                if text.startswith('$'):
                    lat, lon, t = parse_nmea_latlon(text)
                    if lat is not None:
                        return lat, lon, t, text
            else:
                line += c
    return None, None, None, None

def send_location(lat, lon, fix_time, extra=None):
    payload = {
        "api_key": API_KEY,
        "lat": lat,
        "lon": lon,
        "time": fix_time,
        "device_id": "pico-001",
    }
    if extra:
        payload.update(extra)
    try:
        headers = {'Content-Type': 'application/json'}
        r = requests.post(SERVER_URL, json=payload, headers=headers)
        print("POST status:", r.status_code)
        r.close()
    except Exception as e:
        print("send error", e)

def main_loop():
    if not connect_wifi():
        return
    last_hb = 0
    while True:
        # Check SOS
        if sos.value() == 0:  # pressed (active low)
            print("SOS pressed!")
            lat, lon, t, raw = read_gps(timeout=5)
            send_location(lat, lon, t, extra={"event":"SOS", "raw": raw})
            # Simple debounce wait to avoid repeated triggers
            time.sleep(3)
        # periodic heartbeat
        if time.time() - last_hb > HEARTBEAT_SECONDS:
            lat, lon, t, raw = read_gps(timeout=3)
            send_location(lat, lon, t, extra={"event":"heartbeat", "raw": raw})
            last_hb = time.time()
        time.sleep(0.2)

if __name__ == "__main__":
    main_loop()

