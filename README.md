# radio-feed
Private audio streaming box on a Raspberry Pi. USB sound card to Icecast, reachable only over Tailscale.

USB sound card -> Darkice -> Icecast2 -> Tailscale -> browser.

## Before deploying

- Replace `CHANGE_ME` in `config/darkice.cfg` with your Icecast source password. Never commit the real one.
- Set `device =` in `config/darkice.cfg` to your card's name (see the comments in that file).
- Replace `YOUR_USER` in `config/webplayer.service` with your Pi username.
- Do not commit your real `/etc/icecast2/icecast.xml`. It contains the source, relay and admin passwords.

## Layout

- `SETUP.md` step-by-step guide
- `TODO.md` open items, compression is next
- `webplayer/index.html` dark player with spectrum, peak meter, reconnect and stream status. Copy to `~/webplayer/` on the Pi.
- `config/darkice.cfg` goes in `/etc/darkice.cfg`
- `config/darkice.service`, `config/webplayer.service` go in `/etc/systemd/system/`

## Lessons from the build

- Use `127.0.0.1`, not `localhost`. `localhost` resolves to `::1` first and Icecast is not on IPv6.
- USB card numbers can change between reboots (`hw:2,0` became `hw:1,0`). Refer to the card by name instead: `hw:CARD=<name>,DEV=0`.
- Do not bind Icecast to the Tailscale IP. It starts before Tailscale is up and dies. Leave it on all interfaces and restrict with UFW on `tailscale0`.
- Some USB sound cards only do stereo capture. If Darkice says `can't set channels`, use `channel = 2`.
- Keep comments in `darkice.cfg` on their own lines, not after a value.

## Commands

    sudo systemctl status icecast2 darkice webplayer
    sudo journalctl -u darkice -f
    arecord -l
