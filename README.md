# Home Wi‑Fi Monitor

A tiny Python script that:
- Scans connected devices via ARP
- Pings the home gateway to check basic connectivity
- Logs results to CSV and prints a CLI dashboard

## Diagrams
See `/diagrams`:
- Architecture: how data flows between the device, router, and local logging.
- Flowchart: the loop from config → scan → ping → log → print → sleep.

## Quick Start

```bash
python3 --version          # Python 3.8+ recommended
git clone <your-repo-url>
cd home-wifi-monitor
pip3 install -r requirements.txt   # nothing required for minimal version
python3 wifi_monitor.py
