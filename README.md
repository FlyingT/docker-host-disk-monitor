# Monitor for your Docker Host Disk

A lightweight, "quick & dirty" standalone Docker container that monitors your host machine's disk usage and sends an email alert via `swaks` when storage exceeds a defined threshold.

This tool intentionally avoids a Dockerfile. It relies entirely on the `docker-compose.yml` to install dependencies on the fly. It is not considered best practice for production environments, but it is perfect for a quick fix.

No Dockerfile required, uses a slim Debian image. Minimal resource consumption. Just repull the image and restart the container for updates. Use a `.env` file for easy setup.

## How it works

The container spins up a minimal Debian environment and executes a single, continuous Bash script directly from the `docker-compose.yml`. Here is the step-by-step process:

1. As soon as the container starts, it updates the package lists and installs `swaks` (a scriptable SMTP transaction tool) and `procps`.
2. The script enters a continuous `while true` loop to monitor your system.
3. It uses the `df` command to parse the current percentage of used disk space.  
*(Note: By checking `/` inside the container, it effectively monitors the host partition where Docker's virtual filesystem/storage resides, usually the host's main drive).*
4. The current usage is compared against your defined `THRESHOLD`.
5. If the usage exceeds the threshold, `swaks` fires off an SMTP email alert. To prevent spamming your inbox, the script then pauses for the `COOLDOWN_INTERVAL`.
6. If everything is fine, the script simply waits for the `CHECK_INTERVAL` before measuring again.

## Compose

```yaml
version: '3.8'

services:
  disk-monitor:
    image: debian:stable-slim
    container_name: disk-monitor
    init: true
    restart: unless-stopped
    environment:
      - TZ=${TZ}
      - VM_NAME=${VM_NAME}
      - THRESHOLD=${THRESHOLD}
      - MAIL_TO=${MAIL_TO}
      - SMTP_SERVER=${SMTP_SERVER}
      - SMTP_PORT=${SMTP_PORT}
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASS=${SMTP_PASS}
      - MAIL_FROM=${MAIL_FROM}
      - CHECK_INTERVAL=${CHECK_INTERVAL}
      - COOLDOWN_INTERVAL=${COOLDOWN_INTERVAL}
    command: >
      bash -c "
      apt-get update && apt-get install -y swaks procps &&
      echo \"Monitoring started. Checking disk usage...\" &&
      while true; do
        USAGE=\$(df --output=pcent / | tail -1 | tr -dc '0-9');

        if [ \"\$$USAGE\" -ge \"\$$THRESHOLD\" ]; then
          echo \"[$$(date)] WARNING: Disk at $$USAGE% usage. Sending Mail...\"
          swaks --to \"$$MAIL_TO\" --from \"$$MAIL_FROM\" --server \"$$SMTP_SERVER\" --port \"$$SMTP_PORT\" --auth LOGIN --auth-user \"$$SMTP_USER\" --auth-password \"$$SMTP_PASS\" --header \"Subject: Disk Alert: $$VM_NAME\" --body \"Warning: The disk of host '$$VM_NAME' is nearly full! Current usage: $$USAGE%\" --silent;

          echo \"[$$(date)] Mail sent. Waiting for $$COOLDOWN_INTERVAL seconds...\"
          sleep $$COOLDOWN_INTERVAL;
        else
          echo \"[$$(date)] Status OK: $$USAGE% in use.\"
        fi

        sleep $$CHECK_INTERVAL;
      done
      "

```

### Required `.env` File

Create a `.env` file in the same directory as your `docker-compose.yml` to populate the variables:

```env
# General Settings
TZ=Europe/Berlin
VM_NAME=My-Docker-Host
THRESHOLD=90 # Warn when disk usage reaches 90%

# Mail Settings
MAIL_TO=alerts@yourdomain.com
MAIL_FROM=disk-monitor@yourdomain.com
SMTP_SERVER=smtp.yourdomain.com
SMTP_PORT=587
SMTP_USER=your_smtp_username
SMTP_PASS=your_smtp_password

# Intervals (in seconds)
CHECK_INTERVAL=900       # Check every 15 minutes (900 seconds)
COOLDOWN_INTERVAL=43200  # Wait 12 hours (43200 seconds) after sending an alert

```

## ⚠️ Disclaimer

This tool is provided "as is" without any warranty of any kind. Since it runs `apt-get update` every time the container starts, it requires outbound internet access. Ensure you are comfortable with running inline package installations via Docker Compose before deploying.
