#!/usr/bin/env python3
"""
Very Simple Home Wi‑Fi Monitor
- Scans devices via ARP
- Pings default gateway
- (Optional) runs a lightweight 'speed' estimate with ping jitter
- Logs results to CSV
- Prints a small CLI dashboard

Run: python3 wifi_monitor.py
"""

import csv
import os
import socket
import subprocess
import time
from datetime import datetime

# -------- Configuration (edit as needed) --------
SCAN_INTERVAL_SECS = 60        # How often to run (seconds)
CSV_PATH = "wifi_monitor_log.csv"
GATEWAY_IP = "192.168.1.1"     # Change to your router IP if different
# ------------------------------------------------

def run_cmd(cmd: str, timeout: int = 10) -> str:
    """Run a shell command and return stdout text."""
    proc = subprocess.run(
        cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
        text=True, timeout=timeout
    )
    if proc.returncode != 0:
        return ""
    return proc.stdout.strip()

def scan_devices_arp():
    """
    Very simple ARP table read.
    Output format differs by OS; this is a basic parser for common cases.
    """
    out = run_cmd("arp -a")
    devices = []
    for line in out.splitlines():
        line = line.strip()
        if not line:
            continue
        ip = None
        mac = None
        if "(" in line and ")" in line:
            ip = line.split("(")[1].split(")")[0]
        if " at " in line:
            try:
                mac = line.split(" at ")[1].split(" ")[0]
            except Exception:
                pass
        if ip:
            devices.append({"ip": ip, "mac": mac or "unknown"})
    return devices

def ping_host(host: str, count: int = 2, timeout: int = 2):
    """Ping a host and return avg latency (ms) and packet loss (%)."""
    out = run_cmd(f"ping -c {count} -W {timeout} {host}", timeout=timeout*count+2)
    if not out:
        return {"avg_ms": None, "loss_pct": 100.0, "ok": False}
    # Parse loss
    loss = 100.0
    for line in out.splitlines():
        if "packet loss" in line:
            try:
                loss = float(line.split(",")[2].strip().split("%")[0])
            except Exception:
                pass
            break
    # Parse avg
    avg = None
    for line in out.splitlines():
        if "min/avg/max" in line or "min/avg/max/mdev" in line:
            # rtt min/avg/max/mdev = 8.797/10.032/12.078/1.028 ms
            stats = line.split("=")[1].split()[0]
            avg = float(stats.split("/")[1])
            break
    return {"avg_ms": avg, "loss_pct": loss, "ok": loss == 0.0}

def ensure_csv(path: str):
    if not os.path.exists(path):
        with open(path, "w", newline="") as f:
            w = csv.writer(f)
            w.writerow(["timestamp", "host", "metric", "value"])

def log_csv(path: str, rows):
    with open(path, "a", newline="") as f:
        w = csv.writer(f)
        for r in rows:
            w.writerow(r)

def print_dashboard(devices_count, ping_stats):
    ts = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    print("\n" + "="*52)
    print(f"[{ts}] Home Wi‑Fi Monitor")
    print(f"Devices discovered: {devices_count}")
    if ping_stats["ok"]:
        print(f"Gateway {GATEWAY_IP} reachable ✓  avg={ping_stats['avg_ms']} ms, loss={ping_stats['loss_pct']}%")
    else:
        print(f"Gateway {GATEWAY_IP} unreachable ✗  loss={ping_stats['loss_pct']}%")
    print("="*52)

def main():
    ensure_csv(CSV_PATH)
    hostname = socket.gethostname()

    while True:
        ts_iso = datetime.utcnow().isoformat()

        # 1) Scan devices
        devices = scan_devices_arp()
        dev_count = len(devices)

        # 2) Ping gateway
        ping_stats = ping_host(GATEWAY_IP)

        # 3) Log to CSV
        rows = [
            [ts_iso, hostname, "devices_count", dev_count],
            [ts_iso, hostname, "gateway_avg_ms", ping_stats["avg_ms"] if ping_stats["avg_ms"] is not None else ""],
            [ts_iso, hostname, "gateway_loss_pct", ping_stats["loss_pct"]],
            [ts_iso, hostname, "gateway_ok", int(ping_stats["ok"])],
        ]
        log_csv(CSV_PATH, rows)

        # 4) Print dashboard
        print_dashboard(dev_count, ping_stats)

        # 5) Sleep and repeat
        time.sleep(SCAN_INTERVAL_SECS)

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\\nExiting. Bye!")
