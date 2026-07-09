# debian-server-monitor
A lightweight server monitoring tool for Debian that sends real-time alerts 
and status updates directly to your phone via Telegram.

## What is this?
With the help of a Telegram bot and Python scripting, this tool monitors 
your server and sends you instant alerts when something goes wrong. 
It acts like a chatbot that you can also query directly — ask it for 
CPU temperature, disk usage, or service status at any time.

## Features
-  Sends a heartbeat message every 5 minutes confirming the server is running
-  Sends an alert when CPU temperature is too high
-  Sends an alert when a drive is almost full
-  Sends an alert when the server shuts down or reboots
-  Sends an alert when the server comes back online
-  Sends an alert on repeated failed SSH login attempts
-  Sends an alert if Jellyfin goes down

## Requirements
- A server running Debian 12 or later
- A personal computer or phone to receive notifications
- A Telegram account and internet connection
- Basic knowledge of Linux terminal commands

  
## installation
### step 1
> This works on both iOS and Android.
download telegram on your mobile device or your pc
create an account with your phone number and sign in

### step 2 create a bot on your mobile phone
- Search for @BotFather in the search bar
- Send the command /newbot
- Follow the instructions and give your bot a name
- Copy your bot token — keep it safe, never share it publicly

### step 3  Configure credentials
open your terminal and write the comands  
`sudo nano /etc/server-notify.conf`  

in that file write

`TELEGRAM_TOKEN="YOUR_TELEGRAM_TOKEN"
TELEGRAM_CHAT_ID="YOUR_CHAT_ID"`  

>To find your chat ID, send any message to your bot then visit:
https://api.telegram.org/botYOUR_TELEGRAM_TOKEN/getUpdates
Look for "id": inside "chat" — that is your chat ID.


### step 4 install dependencies
```
sudo apt install lm-sensors python3-pip -y
sudo sensors-detect --auto
sudo pip3 install python-telegram-bot --break-system-packages
```

### step 5 Create the monitoring script
Jellyfin check:  
`sudo nano /usr/local/bin/jellyfin-check.sh`  

```
#!/bin/bash
. /etc/server-notify.conf

if ! systemctl is-active --quiet jellyfin; then
  curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage \
    -d chat_id=${TELEGRAM_CHAT_ID} \
    -d text=" Jellyfin is DOWN!"
fi
```

SSH failed login check:  
`sudo nano /usr/local/bin/ssh-fail-check.sh`  

``` 
#!/bin/bash
. /etc/server-notify.conf

FAILS=$(journalctl -u ssh --since "5 minutes ago" | grep -c "Failed password")

if [ "$FAILS" -gt 3 ]; then
  curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage \
    -d chat_id=${TELEGRAM_CHAT_ID} \
    -d text=" ${FAILS} failed SSH login attempts in the last 5 minutes!"
fi
```

Temperature check:  
`sudo nano /usr/local/bin/temp-check.sh`  

```
#!/bin/bash
. /etc/server-notify.conf

TEMP=$(sensors | grep "Package id 0" | awk '{print $4}' | tr -d '+°C')

if (( $(echo "$TEMP > 80" | bc -l) )); then
  curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage \
    -d chat_id=${TELEGRAM_CHAT_ID} \
    -d text=" CPU temperature is HIGH: ${TEMP}°C!"
fi
```

Disk check:  
`sudo nano /usr/local/bin/disk-check.sh`

```
#!/bin/bash
. /etc/server-notify.conf

USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$USAGE" -gt 90 ]; then
  curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage \
    -d chat_id=${TELEGRAM_CHAT_ID} \
    -d text=" Disk almost full: ${USAGE}% used!"
fi
```

Make all scripts executable:  

```
sudo chmod +x /usr/local/bin/jellyfin-check.sh
sudo chmod +x /usr/local/bin/ssh-fail-check.sh
sudo chmod +x /usr/local/bin/temp-check.sh
sudo chmod +x /usr/local/bin/disk-check.sh
```

### step 6 create the Telegram bot script  

`sudo nano /usr/local/bin/server-bot.py`  

Replace YOUR_TELEGRAM_TOKEN and YOUR_CHAT_ID with your real values before running.  

```
#!/usr/bin/env python3
import subprocess
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

TOKEN = "YOUR_TELEGRAM_TOKEN"
ALLOWED_CHAT_ID = YOUR_CHAT_ID

async def temp(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    result = subprocess.run(["sensors"], capture_output=True, text=True)
    temp_line = [l for l in result.stdout.split("\n") if "Package id 0" in l][0]
    await update.message.reply_text(f" {temp_line.strip()}")

async def status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    cpu = subprocess.run("top -bn1 | grep 'Cpu(s)' | awk '{print $2}'", shell=True, capture_output=True, text=True).stdout.strip()
    ram = subprocess.run("free -h | awk '/Mem:/ {print $3\"/\"$2}'", shell=True, capture_output=True, text=True).stdout.strip()
    disk_raw = subprocess.run("df -h | grep -E '^/dev/' | awk '{print $1\" \"$3\"/\"$2\" (\"$5\" used)\"}'", shell=True, capture_output=True, text=True).stdout.strip()
    disk = disk_raw.replace("/dev/sda1", " SSD").replace("/dev/sdb1", " MEDIA")
    temp = subprocess.run("sensors | grep 'Package id 0' | awk '{print $4}'", shell=True, capture_output=True, text=True).stdout.strip()
    uptime = subprocess.run("uptime -p", shell=True, capture_output=True, text=True).stdout.strip()
    jellyfin = subprocess.run(["systemctl", "is-active", "jellyfin"], capture_output=True, text=True).stdout.strip()
    jellyfin_emoji = "" if jellyfin == "active" else ""

    msg = (
        f" Server Status\n\n"
        f" Uptime: {uptime}\n"
        f" CPU: {cpu}%\n"
        f" Temp: {temp}\n"
        f" RAM: {ram}\n\n"
        f" Disks:\n{disk}\n\n"
        f" Jellyfin: {jellyfin_emoji} {jellyfin}"
    )
    await update.message.reply_text(msg)

async def disk(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    result = subprocess.run("df -h", shell=True, capture_output=True, text=True)
    await update.message.reply_text(f" Disk usage:\n\n{result.stdout}")

async def jellyfin(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    result = subprocess.run(["systemctl", "is-active", "jellyfin"], capture_output=True, text=True)
    status = result.stdout.strip()
    emoji = "" if status == "active" else ""
    await update.message.reply_text(f"{emoji} Jellyfin is {status}")

app = ApplicationBuilder().token(TOKEN).build()
app.add_handler(CommandHandler("temp", temp))
app.add_handler(CommandHandler("status", status))
app.add_handler(CommandHandler("disk", disk))
app.add_handler(CommandHandler("jellyfin", jellyfin))

print("Bot is running...")
app.run_polling()
```

### step 7 set up systemd services
Shutdown notification:  
`sudo nano /etc/systemd/system/notify-shutdown.service`  

```
[Unit]
Description=Notify phone on shutdown
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target

[Service]
Type=oneshot
EnvironmentFile=/etc/server-notify.conf
ExecStart=/usr/bin/curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage -d chat_id=${TELEGRAM_CHAT_ID} -d text=" Server is shutting down!"
TimeoutStartSec=10

[Install]
WantedBy=shutdown.target reboot.target halt.target
```

Boot notification:  

`sudo nano /etc/systemd/system/notify-boot.service`  

```
[Unit]
Description=Notify phone on boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
EnvironmentFile=/etc/server-notify.conf
ExecStart=/usr/bin/curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage -d chat_id=${TELEGRAM_CHAT_ID} -d text=" Server is back online!"

[Install]
WantedBy=multi-user.target
```

Telegram bot service:  
`sudo nano /etc/systemd/system/server-bot.service`  

```
[Unit]
Description=Telegram Server Bot
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/server-bot.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable all services:  
```
sudo systemctl enable notify-shutdown.service
sudo systemctl enable notify-boot.service
sudo systemctl enable server-bot.service
sudo systemctl start server-bot.service
```

### step 8 Set up cron jobs
`crontab -e`  

```
*/5 * * * * . /etc/server-notify.conf && curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage -d chat_id=${TELEGRAM_CHAT_ID} -d text=" Heartbeat OK" > /dev/null 2>&1
*/5 * * * * /usr/local/bin/jellyfin-check.sh
*/5 * * * * /usr/local/bin/ssh-fail-check.sh
*/5 * * * * /usr/local/bin/temp-check.sh
0 * * * * /usr/local/bin/disk-check.sh
```



## configuration
After installation, there are a few things you need to customize for your own setup:

**Telegram credentials** — add your own token and chat ID in:
`/etc/server-notify.conf`

**Drive names** — in `server-bot.py`, replace `/dev/sda1` and `/dev/sdb1` 
with your own drive names. You can find yours by running:
`df -h`

**Temperature threshold** — in `temp-check.sh`, the default alert triggers 
at 80°C. Change this to suit your hardware.

**Disk threshold** — in `disk-check.sh`, the default alert triggers at 90% 
usage. Change this to your preference.

## Usage
Send these commands directly to your Telegram bot:

| Command | Description |
|---|---|
| `/status` | Full server overview — CPU, RAM, temperature, disks, Jellyfin |
| `/temp` | Current CPU temperature |
| `/disk` | Full disk usage for all drives |
| `/jellyfin` | Check if Jellyfin is running |

## file Structure
```
/etc/server-notify.conf          # Stores the Telegram credentials  
/usr/local/bin/jellyfin-check.sh # Checks if Jellyfin is still running  
/usr/local/bin/ssh-fail-check.sh # Monitors failed login attempts via SSH  
/usr/local/bin/temp-check.sh     # Checks the CPU temperature and alerts if too high  
/usr/local/bin/disk-check.sh     # Checks drive usage and alerts if almost full  
/usr/local/bin/server-bot.py     # Main Python script for the Telegram bot  
/etc/systemd/system/notify-shutdown.service  # Systemd service that sends a message when the server shuts down  
/etc/systemd/system/notify-boot.service      # Systemd service that sends a message when the server starts up  
/etc/systemd/system/server-bot.service       # Systemd service that keeps the Telegram bot running at all times  
```
## contributing
Pull requests are welcome! If you have ideas for new alerts or commands, feel free to open an issue.

## License
MIT License


