# Monitor for your Docker Host Disk

A lightweight, "quick & dirty" standalone Docker container that monitors your host machine's disk usage and sends an email alert via `swaks` when storage exceeds a defined threshold.  

This tool intentionally avoids a Dockerfile. It relies entirely on the docker-compose.yml to install dependencies on the fly. It is not considered best practice for production environments, but it is perfect for a "set it and forget it" quick fix.

## Features
- **Zero Maintenance**: No Dockerfile required, uses a slim Debian image.
- **Low Footprint**: Minimal resource consumption.
- **Flexible Configuration**: Use a `.env` file for easy setup or rely on defaults.
- **Smart Fallbacks**: If you don't provide a `.env`, it works with settings directly in Compose.

## How it works
The container mounts your host's root directory (`/`) as read-only into `/host`. It periodically runs a shell loop that checks the disk percentage. If the threshold is hit, it uses `swaks` (Simple Mail Transfer Protocol and Email Transaction Exchanger) to fire off an email.

## Quick Start with Compose, bring your own .env or edit the placeholders

```yaml
version: '3.8'

services:
  disk-monitor:
    image: debian:stable-slim
    container_name: disk-monitor
    init: true
    restart: unless-stopped
    volumes:
      - /:/host:ro
    environment:
      - TZ=Europe/Berlin # set your timezone
      - VM_NAME=HOSTNAME # set hostname that should be mentioned in the mail
      - THRESHOLD=90 # set at which percentage of used storage you want to trigger the alert
      - MAIL_TO=your@mail.com # set the recipient 
      - SMTP_SERVER=mail.server.com # address of the mailserver
      - SMTP_PORT=25 # port for SMTP - usually 25, 465 or 587
      - SMTP_USER=alert # username or mail of the account that sends the alert
      - SMTP_PASS=XXXXXXX # password of the account that sends the alert
      - MAIL_FROM=alert@mail.com # mail of the account that sends the alert
      - CHECK_INTERVAL=900 # how often the below check should run in seconds
      - COOLDOWN_INTERVAL=43200  # how long to wait after the last alert in seconds
    command: >
      bash -c "
      apt-get update && apt-get install -y swaks &&
      echo \"Monitoring started. Checking /host for usage > $$THRESHOLD%...\" &&
      while true; do
        USAGE=\$(df --output=pcent /host | tail -1 | tr -dc '0-9');
        
        if [ \"$$USAGE\" -ge \"$$THRESHOLD\" ]; then
          echo \"[$(date)] WARNING: Disk at $$USAGE% usage. Sending Mail...\"
          swaks --to \"$$MAIL_TO\" --from \"$$MAIL_FROM\" --server \"$$SMTP_SERVER\" --port \"$$SMTP_PORT\" --auth LOGIN --auth-user \"$$SMTP_USER\" --auth-password \"$$SMTP_PASS\" --header \"Subject: Disk Alert: $$VM_NAME\" --body \"Warning: The disk of host '$$VM_NAME' is nearly full! Current usage: $$USAGE%\" --silent;
          
          echo \"[$(date)] Mail send. Waiting for $$COOLDOWN_INTERVAL seconds until next check...\"
          sleep $$COOLDOWN_INTERVAL;
        else
          echo \"[$(date)] Status OK: $$USAGE% in use.\"
        fi
        
        sleep $$CHECK_INTERVAL;
      done
      "
```

## ⚠️ Disclaimer
This tool is provided "as is" without any warranty of any kind. Since it runs apt-get update every time the container starts, it requires internet access. It mounts your host filesystem as read-only for safety, but always use caution when mounting system directories.
