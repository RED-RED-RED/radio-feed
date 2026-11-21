# radio-feed
Private audio streaming box on a Raspberry Pi. USB sound card to Icecast, reachable only over Tailscale.

USB sound card -> Darkice -> Icecast2 -> Tailscale -> browser.

## Layout

- `SETUP.md` step-by-step guide
- `TODO.md` open items, compression is next
- `webplayer/index.html` dark/tactical player with visualizer and status. Copy to `~/webplayer/` on the Pi.
- `config/darkice.cfg` goes in `/etc/darkice.cfg`. Set the real source password from `/etc/icecast2/icecast.xml`.
- `config/darkice.service`, `config/webplayer.service` go in `/etc/systemd/system/`

## Lessons from the build

- Darkice does not like trailing comments on a config line. Keep values bare.
- Use `127.0.0.1`, not `localhost`. `localhost` resolves to `::1` first and Icecast is not on IPv6.
- USB card index changes between reboots (was `hw:2,0`, became `hw:1,0`). Check `arecord -l`. Better fix is on the TODO list.
- Do not bind Icecast to the Tailscale IP. It starts before Tailscale is up and dies. Leave it on all interfaces and restrict with UFW on `tailscale0`.
- The Sound Blaster X-Fi Go does not support mono capture. Keep `channel = 2`.
- The `darkice.service` here has a 10 second pre-start delay to give the USB card time to enumerate.

## Commands

    sudo systemctl status icecast2 darkice webplayer
    sudo journalctl -u darkice -f
    arecord -l

## Before deploying

- Replace `CHANGE_ME` in `config/darkice.cfg` with your Icecast source password. Never commit the real one.
- Replace `YOUR_USER` in `config/webplayer.service` with your Pi username.
- Do not commit your real `/etc/icecast2/icecast.xml`. It contains the source, relay and admin passwords.
