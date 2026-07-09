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
-  List all running services
-  Stop/start/restart any service via chat
-  View live service logs
-  Natural language service control — just type "stop jellyfin" or "show nginx logs"
-  /help command showing all available commands

## Requirements
- A server running Debian 12 or later
- A personal computer or phone to receive notifications
- A Telegram account and internet connection
- Basic knowledge of Linux terminal commands
- Ollama running locally with Mistral model installed


  
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


### step 4a install dependencies
```
sudo apt install lm-sensors python3-pip -y
sudo sensors-detect --auto
sudo pip3 install python-telegram-bot --break-system-packages
```
### step 4b install Ollama and mistral

```
curl -fsSL https://ollama.com/install.sh | sh
ollama pull mistral
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
import requests
import json
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, filters

TOKEN = "YOUR_TELEGRAM_TOKEN"
ALLOWED_CHAT_ID = YOUR_CHAT_ID

# ── Server functions ──────────────────────────────────────

def get_temp():
    result = subprocess.run("sensors | grep 'Package id 0' | awk '{print $4}'", shell=True, capture_output=True, text=True)
    return f" CPU Temperature: {result.stdout.strip()}"

def get_status():
    cpu = subprocess.run("top -bn1 | grep 'Cpu(s)' | awk '{print $2}'", shell=True, capture_output=True, text=True).stdout.strip()
    ram = subprocess.run("free -h | awk '/Mem:/ {print $3\"/\"$2}'", shell=True, capture_output=True, text=True).stdout.strip()
    disk_raw = subprocess.run("df -h | grep -E '^/dev/' | awk '{print $1\" \"$3\"/\"$2\" (\"$5\" used)\"}'", shell=True, capture_output=True, text=True).stdout.strip()
    disk = disk_raw.replace("/dev/sda1", " SSD").replace("/dev/sdb1", " MEDIA")
    temp = subprocess.run("sensors | grep 'Package id 0' | awk '{print $4}'", shell=True, capture_output=True, text=True).stdout.strip()
    uptime = subprocess.run("uptime -p", shell=True, capture_output=True, text=True).stdout.strip()
    jellyfin = subprocess.run(["systemctl", "is-active", "jellyfin"], capture_output=True, text=True).stdout.strip()
    jellyfin_emoji = "" if jellyfin == "active" else ""
    return (
        f" Server Status\n\n"
        f" Uptime: {uptime}\n"
        f" CPU: {cpu}%\n"
        f" Temp: {temp}\n"
        f" RAM: {ram}\n\n"
        f" Disks:\n{disk}\n\n"
        f" Jellyfin: {jellyfin_emoji} {jellyfin}"
    )

def get_disk():
    result = subprocess.run("df -h | grep -E '^/dev/'", shell=True, capture_output=True, text=True)
    disk = result.stdout.replace("/dev/sda1", " SSD").replace("/dev/sdb1", " MEDIA")
    return f" Disk usage:\n\n{disk}"

def get_jellyfin():
    result = subprocess.run(["systemctl", "is-active", "jellyfin"], capture_output=True, text=True)
    status = result.stdout.strip()
    emoji = "" if status == "active" else ""
    return f"{emoji} Jellyfin is {status}"

def get_ram():
    result = subprocess.run("free -h", shell=True, capture_output=True, text=True)
    return f" RAM usage:\n\n{result.stdout}"

def get_uptime():
    result = subprocess.run("uptime -p", shell=True, capture_output=True, text=True)
    return f" Uptime: {result.stdout.strip()}"

def get_services():
    result = subprocess.run(
        "systemctl list-units --type=service --state=running --no-pager --no-legend | awk '{print $1}'",
        shell=True, capture_output=True, text=True
    )
    services = result.stdout.strip()
    return f" Running services:\n\n{services}"

def manage_service(action, service_name):
    # Safety check — block dangerous commands
    BLOCKED = ["sshd", "ssh", "networking", "systemd", "dbus", "cron"]
    if any(blocked in service_name.lower() for blocked in BLOCKED):
        return f" Cannot {action} {service_name} — this service is protected."

    result = subprocess.run(
        ["systemctl", action, service_name],
        capture_output=True, text=True
    )
    if result.returncode == 0:
        return f" Successfully ran '{action}' on {service_name}"
    else:
        return f" Failed to {action} {service_name}:\n{result.stderr.strip()}"

def get_service_logs(service_name):
    result = subprocess.run(
        f"journalctl -u {service_name} -n 20 --no-pager",
        shell=True, capture_output=True, text=True
    )
    return f" Last 20 logs for {service_name}:\n\n{result.stdout.strip()}"

def get_help():
    return (
        " Here is everything I can do:\n\n"
        " *Server Info*\n"
        "/status — Full server overview\n"
        "/temp — CPU temperature\n"
        "/disk — Disk usage\n"
        "/ram — RAM usage\n"
        "/uptime — How long the server has been running\n\n"
        " *Services*\n"
        "/jellyfin — Check if Jellyfin is running\n"
        "/services — List all running services\n"
        "/start <name> — Start a service\n"
        "/stop <name> — Stop a service\n"
        "/restart <name> — Restart a service\n"
        "/logs <name> — Show last 20 logs of a service\n\n"
        " *Natural Language*\n"
        "You can also just ask me normally, for example:\n"
        "• 'is everything ok?'\n"
        "• 'how hot is my cpu?'\n"
        "• 'is jellyfin working?'\n"
        "• 'how much disk space is left?'\n\n"
        " *Help*\n"
        "/help — Show this message"
    )
# ── Ollama natural language handler ───────────────────────

def ask_ollama(user_message):
    prompt = f"""You are a server assistant. Based on the user's message, decide which action to take.
Only respond with one of these exact formats, nothing else:

- status
- temp
- disk
- jellyfin
- ram
- uptime
- help
- services
- stop <service-name>
- start <service-name>
- restart <service-name>
- logs <service-name>
- unknown

Examples:
"stop zerotier" → stop zerotier-one.service
"restart jellyfin" → restart jellyfin.service
"show me the logs for nginx" → logs nginx.service
"start ollama" → start ollama.service
"how hot is my cpu" → temp
"is everything ok" → status

Always add .service at the end of service names.

User message: "{user_message}"
"""
    response = requests.post("http://localhost:11434/api/generate", json={
        "model": "mistral",
        "prompt": prompt,
        "stream": False
    })
    result = response.json()["response"].strip().lower()

    for keyword in ["status", "temp", "disk", "jellyfin", "ram", "uptime", "help", "services"]:
        if result.startswith(keyword):
            return keyword

    for action in ["stop", "start", "restart", "logs"]:
        if result.startswith(action):
            parts = result.split()
            if len(parts) >= 2:
                return f"{action} {parts[1]}"

    return "unknown"

# ── Telegram handlers ─────────────────────────────────────

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return

    user_message = update.message.text
    await update.message.reply_text(" Thinking...")

    action = ask_ollama(user_message)
    parts = action.split()

    if action == "status":
        await update.message.reply_text(get_status())
    elif action == "temp":
        await update.message.reply_text(get_temp())
    elif action == "disk":
        await update.message.reply_text(get_disk())
    elif action == "jellyfin":
        await update.message.reply_text(get_jellyfin())
    elif action == "ram":
        await update.message.reply_text(get_ram())
    elif action == "uptime":
        await update.message.reply_text(get_uptime())
    elif action == "help":
        await update.message.reply_text(get_help(), parse_mode="Markdown")
    elif action == "services":
        await update.message.reply_text(get_services())
    elif len(parts) == 2 and parts[0] in ["stop", "start", "restart"]:
        await update.message.reply_text(manage_service(parts[0], parts[1]))
    elif len(parts) == 2 and parts[0] == "logs":
        await update.message.reply_text(get_service_logs(parts[1]))
    else:
        await update.message.reply_text(
            " I didn't understand that. Type /help to see everything I can do."
        )
# ── Command handlers (keep the old ones working too) ──────

async def cmd_temp(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    await update.message.reply_text(get_temp())

async def cmd_status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    await update.message.reply_text(get_status())

async def cmd_disk(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    await update.message.reply_text(get_disk())

async def cmd_jellyfin(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    await update.message.reply_text(get_jellyfin())

async def cmd_ram(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    await update.message.reply_text(get_ram())

async def cmd_uptime(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    await update.message.reply_text(get_uptime())

async def cmd_services(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    await update.message.reply_text(get_services())

async def cmd_stop(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    if not context.args:
        await update.message.reply_text("Usage: /stop <service-name>\nExample: /stop jellyfin")
        return
    service = context.args[0]
    await update.message.reply_text(manage_service("stop", service))

async def cmd_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    if not context.args:
        await update.message.reply_text("Usage: /start <service-name>\nExample: /start jellyfin")
        return
    service = context.args[0]
    await update.message.reply_text(manage_service("start", service))

async def cmd_restart(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    if not context.args:
        await update.message.reply_text("Usage: /restart <service-name>\nExample: /restart jellyfin")
        return
    service = context.args[0]
    await update.message.reply_text(manage_service("restart", service))

async def cmd_logs(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    if not context.args:
        await update.message.reply_text("Usage: /logs <service-name>\nExample: /logs jellyfin")
        return
    service = context.args[0]
    await update.message.reply_text(get_service_logs(service))

async def cmd_help(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ALLOWED_CHAT_ID:
        return
    await update.message.reply_text(get_help(), parse_mode="Markdown")
# ── Start bot ─────────────────────────────────────────────

app = ApplicationBuilder().token(TOKEN).build()

# Slash commands still work
app.add_handler(CommandHandler("services", cmd_services))
app.add_handler(CommandHandler("stop", cmd_stop))
app.add_handler(CommandHandler("start", cmd_start))
app.add_handler(CommandHandler("restart", cmd_restart))
app.add_handler(CommandHandler("logs", cmd_logs))
app.add_handler(CommandHandler("temp", cmd_temp))
app.add_handler(CommandHandler("status", cmd_status))
app.add_handler(CommandHandler("disk", cmd_disk))
app.add_handler(CommandHandler("jellyfin", cmd_jellyfin))
app.add_handler(CommandHandler("help", cmd_help))
app.add_handler(CommandHandler("ram", cmd_ram))
app.add_handler(CommandHandler("uptime", cmd_uptime))


# Natural language for normal messages
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

print("Bot is running with Ollama natural language support...")
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
| `/services` | List all running services |
| `/start <name>` | Start a service |
| `/stop <name>` | Stop a service |
| `/restart <name>` | Restart a service |
| `/logs <name>` | Show last 20 logs of a service |
| `/help` |  Show all available commands |


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


