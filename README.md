# Woman Safety — GPS SOS Tracker (Raspberry Pi Pico W + NEO-6M + R backend)
A compact real-time location tracking and emergency alert system built with Raspberry Pi Pico W and NEO-6M GPS module.
Designed to help women send instant SOS alerts, share live GPS location, and notify family or authorities during dangerous or life-threatening situations.

🔥 Key Features


🔴 Emergency SOS Button

With a single press, the device sends instant alerts to predefined contacts.

Delivers real-time data such as GPS coordinates and timestamp.

📡 Live GPS Tracking

Continuously tracks the user’s location using the NEO-6M GPS module.

Sends periodic updates to the backend for live map monitoring.

🌐 Wi-Fi Enabled Communication

Uses Raspberry Pi Pico W for reliable wireless transmission.

Sends location data to a secure API or dashboard.

📍 Real-Time Dashboard (R + Shiny)

Backend built using R (Plumber API) to receive device data.

Interactive dashboard shows live movement, SOS events, and history logs.

📨 Automatic Alerts

In emergency mode, the system can automatically notify:

Family members

Police or security units

Community emergency channels

Alerts can be sent via SMS, WhatsApp, or email.

🛡️ Safety in Critical Situations

Made to assist women facing:

Harassment

Threats or stalking

Abduction

Sexual assault

Any unsafe situation where immediate help is needed

This device ensures quick communication, precise location sharing, and fast action during emergencies.

🧩 Hardware Overview

Raspberry Pi Pico W

NEO-6M GPS module

Physical SOS button

Rechargeable battery

Portable enclosure for safety & mobility

⚙️ System Workflow

GPS module acquires real-time coordinates.

SOS button pressed → Emergency mode activated.

Device sends location + alert to server (via Wi-Fi).

R Backend stores the data and triggers notifications.

Dashboard displays live updates for responders.

🎯 Why This Project Matters

This system bridges the gap between danger and help, offering women a fast and dependable way to communicate distress.
With real-time tracking and instant alerts, it increases the possibility of early intervention, quick rescue, and preventing further harm.
