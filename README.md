# InstallingZigbee2MqttAndOpenHabOnAndroid
A guide how to make an zigbee2mqtt and openhab work with an android phone
## ✅ Full Guide: Zigbee2MQTT + TCPUART + Home Assistant Core on Android (Termux)

### 🔧 Requirements

* Rooted Android phone (not so sure about it, try it and let me know)
* [Termux (from F-Droid)](https://f-droid.org/en/packages/com.termux/)
* [TCPUART app](https://play.google.com/store/apps/details?id=com.hardcodedjoy.tcpuart)
* Zigbee USB dongle (e.g. Sonoff ZBDongle-E)
* USB OTG cable (if needed)
* Internet access
* Device architecture: **aarch64** (required for HA install script)

---

### ✅ Step 1: Set up Termux & Install Base Packages

```bash
pkg update && pkg upgrade -y
pkg install python git nodejs termux-services
pkg install libffi openssl clang make
pip install --upgrade pip
npm install -g pnpm yarn
```

---

### ✅ Step 2: Configure USB-to-TCP with TCPUART

1. Launch the **TCPUART** app
2. Choose your Zigbee USB dongle
3. Set TCP port to **2323**
4. Enable TCP server mode
5. Leave the app running in the background

---

### ✅ Step 3: Install Zigbee2MQTT via `pnpm`

```bash
cd ~
git clone https://github.com/Koenkk/zigbee2mqtt.git
cd zigbee2mqtt
pnpm install
```

---

### ✅ Step 4: Configure Zigbee2MQTT

```bash
mkdir -p data
nano data/configuration.yaml
```

Paste this config:

```yaml
serial:
  port: tcp://127.0.0.1:2323
  adapter: ember

mqtt:
  base_topic: zigbee2mqtt
  server: 'mqtt://localhost:1883'

frontend:
  port: 8081
```

Save with `Ctrl+O`, `Enter`, then `Ctrl+X`.

---

### ✅ Step 5: Install MQTT Broker (Mosquitto)

```bash
pkg install mosquitto
mkdir -p ~/.mosquitto
echo -e "listener 1883\nallow_anonymous true" > ~/.mosquitto/mosquitto.conf
```

Start Mosquitto:

```bash
mosquitto -c ~/.mosquitto/mosquitto.conf &
```

Verify it's running:

```bash
netstat -tuln | grep 1883
```

---

### ✅ Step 6: Start Zigbee2MQTT

```bash
cd ~/zigbee2mqtt
pnpm start
```

You should see:

```
Zigbee2MQTT started!
Connected to MQTT server at mqtt://localhost:1883
```

Frontend is accessible at:
**[http://127.0.0.1:8081](http://127.0.0.1:8081)**

---


### ✅ Step 7: Install OpenHab on Ubuntu

---

### ✅ What to do:

1. Installs essential Termux packages. Installs Ubuntu via `proot-distro`. Logs into Ubuntu

```bash
pkg update -y && pkg install -y proot-distro git python screen && proot-distro install ubuntu && proot-distro login ubuntu
```

2. Install preconditions for OpenHab 5:

```bash
add-apt-repository universe
apt update
apt install openjdk-21-jdk
```

3. Install OpenHab 5 (described here or take the code below: https://www.openhab.org/download/)

```bash
curl -fsSL "https://openhab.jfrog.io/artifactory/api/gpg/key/public" | gpg --dearmor > openhab.gpg
mkdir /usr/share/keyrings
mv openhab.gpg /usr/share/keyrings
chmod u=rw,g=r,o=r /usr/share/keyrings/openhab.gpg

apt-get install apt-transport-https
echo 'deb [signed-by=/usr/share/keyrings/openhab.gpg] https://openhab.jfrog.io/artifactory/openhab-linuxpkg stable main' | sudo tee /etc/apt/sources.list.d/openhab.list

apt-get update && apt-get install openhab
apt-get install openhab-addons
```

4. Service-Stuff will fail since no systemctl stuff is existing in termux proot distros, thats why we start it on our own, with 

```bash
/usr/share/openhab# ./start.sh
```

---

### ✅ Step 8: Autostart everything with screen

Enable wake lock to prevent Android from killing background processes:

```bash
termux-wake-lock
```

Create an autostart script:

```bash
nano ~/start_automation.sh
```

Paste:

```bash
#!/data/data/com.termux/files/usr/bin/bash

# Start a new screen session in detached mode
screen -dmS iotsetup

# Window 0: Mosquitto + Zigbee2MQTT
screen -S iotsetup -p 0 -X stuff 'echo "[Starting Mosquitto + Zigbee2MQTT]" && mosquitto -c ~/.mosquitto/mosquitto.conf &\n'
screen -S iotsetup -p 0 -X stuff 'cd ~/zigbee2mqtt && pnpm start\n'

# Create Window 1: Ubuntu + OpenHAB
screen -S iotsetup -X screen -t openhab
screen -S iotsetup -p 1 -X stuff 'echo "[Logging into Ubuntu and starting OpenHAB]" && proot-distro login ubuntu --shared-tmp -- bash -c "cd /usr/share/openhab && ./start.sh"\n'

# Window 2: Console with system status message
screen -S iotsetup -X screen -t console
screen -S iotsetup -p 2 -X stuff 'clear && echo -e "\n=== IOT SYSTEM STARTED ===\n[0] Mosquitto + Zigbee2MQTT\n[1] Ubuntu + OpenHAB\n[2] Console (you are here)\n==========================\n" && exec bash\n'

# Attach to the session and go to console window
screen -r iotsetup -p 2
```

Save and make executable:

```bash
chmod +x ~/start_automation.sh
```

Add to `.bash_profile` so termux trys to connect on startup to the existing screen that we created with out start_automation.sh script

```bash
if command -v screen >/dev/null && ! screen -list | grep -q "Attached"; then
  if screen -list | grep -q "iotsetup"; then
    screen -r iotsetup
  fi
fi
```

Start the whole automation with this:

```bash
./start_automation.sh
```

From now on Mosquitto + Zigbee2MQTT and Ubuntu + openhab should run on Ctrl-A + Ctrl-0 and Ctrl-A + Ctrl-1. You are then in a console on Ctrl-A + Ctrl-2. 
Hint: Be careful don´t try to write exit or something to exit your ssh. Just close the window, on reconnect you will be back in your iot session.

---
