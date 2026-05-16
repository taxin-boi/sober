# 🛡️ Sober Watchdog v2.0

> **Enforce your focus.** A lightweight, automated session manager for [Sober](https://sober.vinegarhq.org/) on Linux.

Sober Watchdog monitors your gameplay in the background and enforces a strict daily time limit. Once your time is up, the game is gracefully closed (and then forced closed) to ensure you stay on track.

---

## ✨ Features

* 📅 **Daily Persistence:** Tracks usage across multiple sessions; resets automatically at midnight.
* 🚀 **Zero Overhead:** Written in pure Bash with a 10s polling interval—uses 0% CPU.
* 🔔 **Smart Notifications:** Sends a 5-minute warning and a final termination alert via `libnotify`.
* 📦 **Universal:** Works for both **Flatpak** and **Native** installations of Sober.
* 🛠️ **Systemd Powered:** Runs as a user-level service that starts automatically on login.

---

## 🛠️ Installation

### 1. The Watchdog Script
Create the script in your home directory:

```bash
nvim ~/sr-watchdog.sh
```

```bash
#!/bin/bash

# Configuration
LIMIT_MINUTES=60
DATA_FILE="$HOME/.sober_usage"
TODAY=$(date +%Y-%m-%d)
APP_ID="org.sober.Sober"

# Initialize daily file
if [ ! -f "$DATA_FILE" ] || [ "$(head -n 1 "$DATA_FILE")" != "$TODAY" ]; then
    echo "$TODAY" > "$DATA_FILE"
    echo "0" >> "$DATA_FILE"
fi

while true; do
    # Detection logic confirmed working by user
    if pgrep -f "bwrap.*sober" > /dev/null; then
        
        USAGE=$(sed -n '2p' "$DATA_FILE")
        
        if [ "$USAGE" -ge "$LIMIT_MINUTES" ]; then
            # systemd needs these for notifications
            export DISPLAY=:0
            export DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u)/bus
            
            notify-send "Time's Up!" "Daily 10m limit reached. Closing Sober." -u critical
            
            # --- THE FORCE KILL SEQUENCE ---
            # 1. Official Flatpak kill
            flatpak kill "$APP_ID" 2>/dev/null
            
            # 2. Force kill Bubblewrap process (-9 is unblockable)
            pkill -9 -f "bwrap.*sober" 2>/dev/null
            
            # 3. Cleanup kill for anything 'sober'
            pkill -9 -fi "sober" 2>/dev/null
        else
            sleep 60
            USAGE=$((USAGE + 1))
            sed -i "2s/.*/$USAGE/" "$DATA_FILE"
        fi
    else
        # Game closed, check again in 30s
        sleep 30
    fi
done
```

---

### 2. Make it Executable

```bash
chmod +x ~/sr-watchdog.sh
```

---

### 3. Setup the Systemd Service
This ensures the script runs in the background without needing a terminal open.

```bash
mkdir -p ~/.config/systemd/user/
nvim ~/.config/systemd/user/sr-limit.service
```

```Ini, TOML
[Unit]
Description=Sober Daily Limit Watchdog
After=graphical-session.target

[Service]
ExecStart=%h/sr-watchdog.sh
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
```

---

### 4. Activation
Enable and start the service:

```bash
systemctl --user daemon-reload
systemctl --user enable --now sr-limit.service
```

Check the status to ensure it's active:

```bash
systemctl --user status sr-limit.service
```
reboot

---

### 5. Check Your Usage
Run this for a quick status update:

```bash
watch -n 5 "cat ~/.sober_usage"
```

---

## 🤝 Contributing
Feel free to fork this and add more features like weekly limits or GUI trackers.

### Stay sober, stay productive.
