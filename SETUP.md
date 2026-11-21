# Pi 3B Radio Stream Setup Guide

Private audio streaming via Tailscale using Icecast2 and Darkice.

```
Radio -> 3.5mm -> USB Sound Card -> Pi 3B -> Ethernet
                                        |
                                    Icecast2 (port 8000)
                                        |
                                    Tailscale
                                        |
                          http://radio-pi:8000/radio
                          (browser, VLC, any device on your tailnet)
```

## What you need

- Raspberry Pi 3 Model B
- MicroSD card (16GB+, a high-endurance card is best for 24/7 use)
- USB sound card with a line-in or mic input
- 3.5mm cable from the radio to the sound card
- Ethernet cable
- 5V/2.5A power supply

## Step 1: Flash Pi OS Lite

Use Raspberry Pi Imager:

1. Choose Raspberry Pi OS Lite (64-bit). No desktop needed.
2. Open the settings (gear icon) before writing and set:
   - Hostname: `radio-pi`
   - Enable SSH
   - Username and password
3. Flash the card, insert it, connect ethernet, power on.

## Step 2: SSH in and update

```bash
ssh <your-user>@radio-pi.local
sudo apt update && sudo apt upgrade -y
```

If `.local` does not resolve, find the Pi's IP in your router's DHCP table.

## Step 3: Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the auth link it prints. Then:

```bash
tailscale ip -4
```

Use that IP, or the MagicDNS hostname if MagicDNS is enabled. In the Tailscale admin console, disable key expiry for this device so it does not drop off after 90 days.

## Step 4: Identify the USB sound card

Plug it in, then list **capture** devices:

```bash
arecord -l
```

Example output:

```
card 1: Device [USB Audio Device], device 0: USB Audio [USB Audio]
```

The number can change between reboots when USB devices enumerate in a different order, so use the card **name** instead. Here the name is `Device`, which gives:

```
hw:CARD=Device,DEV=0
```

Test capture:

```bash
arecord -D hw:CARD=Device,DEV=0 -f cd -t wav -d 5 test.wav && aplay test.wav
```

## Step 5: Install Icecast2 and Darkice

```bash
sudo apt install -y icecast2 darkice
```

In the Icecast2 install dialog:
- Configure Icecast2 automatically: Yes
- Hostname: `localhost`
- Set the source, relay and admin passwords and write them down.

You can look the source password up later with:

```bash
sudo grep source-password /etc/icecast2/icecast.xml
```

## Step 6: Configure Darkice

```bash
sudo nano /etc/darkice.cfg
```

Use the template in `config/darkice.cfg`. Things that matter:

- Put comments on their own line. Keep values bare, with nothing after them.
- Use `server = 127.0.0.1`, not `localhost`. `localhost` can resolve to IPv6 first and Icecast is not listening there.
- `password` must match the Icecast source password exactly.
- `device` must match the card name from Step 4.
- Some sound cards only support stereo capture. If you get `can't set channels`, keep `channel = 2`.
- `realtime = no` is there because Darkice will run as a normal user in Step 8.

Darkice reads the Icecast password from this file, so lock it down once the service user exists (Step 8).

## Step 7: Start Icecast2

```bash
sudo systemctl enable icecast2
sudo systemctl start icecast2
ss -tlnp | grep 8000
```

Leave Icecast listening on all interfaces and restrict access with the firewall in Step 11. Binding it to the Tailscale IP in `icecast.xml` fails at boot because Icecast starts before Tailscale is up.

This box only needs a couple of listeners. In `/etc/icecast2/icecast.xml`, under `<limits>`, lower the defaults:

```xml
<clients>5</clients>
<sources>1</sources>
```

Then `sudo systemctl restart icecast2`.

Test Darkice by hand first:

```bash
sudo darkice -c /etc/darkice.cfg
```

It should run with no errors. Press Ctrl+C when done.

## Step 8: Darkice systemd service

Darkice does not need root. Create a service user in the `audio` group and let it read the config:

```bash
sudo useradd -r -G audio -s /usr/sbin/nologin darkice
sudo chown root:darkice /etc/darkice.cfg
sudo chmod 640 /etc/darkice.cfg
```

Copy `config/darkice.service` to `/etc/systemd/system/darkice.service`, then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable darkice
sudo systemctl start darkice
sudo systemctl status icecast2 darkice
```

If the USB card is not ready when the service first starts, Darkice exits and systemd retries every 5 seconds until it is.

## Step 9: Listen

From any device on your tailnet:

```
http://radio-pi:8000/radio
```

Works in a browser, VLC (Media > Open Network Stream), and most apps that play HTTP audio streams. The Icecast admin panel is at `http://radio-pi:8000/admin`.

## Step 10: Web player (optional)

Copy `webplayer/index.html` to `~/webplayer/` on the Pi. It builds the stream URL from the host you loaded the page from, so no editing is needed.

Install `config/webplayer.service` (replace `YOUR_USER` with your username), then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable webplayer
sudo systemctl start webplayer
```

Open `http://radio-pi:8080`.

## Step 11: Restrict access to Tailscale only

```bash
sudo apt install -y ufw
sudo ufw allow ssh
sudo ufw allow in on tailscale0 to any port 8000
sudo ufw allow in on tailscale0 to any port 8080
sudo ufw enable
```

Port 8000 and 8080 are now only reachable over the Tailscale interface.

## Capture volume

USB adapters often default to a bad capture level. Open the mixer straight into capture view:

```bash
alsamixer -c 1 -V capture
```

Adjust the level, then save it:

```bash
sudo alsactl store
```

If the signal is distorted, check that the radio is going into **line in** and not a mic input, and lower the capture gain.

## Troubleshooting

| Problem | Fix |
|---|---|
| Darkice `TcpSocket connect error [111]` | Icecast is not running or not reachable. Check `ss -tlnp \| grep 8000`. Use `127.0.0.1` in `server =`. |
| Darkice `can't open connector [0]` | Usually the audio device. Run `arecord -l` and make sure `device =` matches the current card number. |
| Darkice `can't set channels` | The card does not support that channel count. Use `channel = 2`. |
| Works until reboot, then stops | You are probably using a card number. Switch `device =` to the card name (`hw:CARD=<name>,DEV=0`). |
| Darkice fails to start as the `darkice` user | Check `journalctl -u darkice`. Confirm the user is in the `audio` group and can read `/etc/darkice.cfg`. |
| Stream URL gives 404 | Darkice has not connected to Icecast yet. Check `journalctl -u darkice -f`. |
| Silent or very quiet audio | Raise capture gain in `alsamixer`. |
| Distorted audio | Lower capture gain, use line in rather than mic in. |
| Cannot reach from another device | Check `tailscale status` on both ends and the UFW rules. |

## Service commands

```bash
sudo systemctl status icecast2 darkice webplayer
sudo systemctl restart icecast2 darkice
journalctl -u darkice -f
```
