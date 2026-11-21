# TODO

## Compression (next up)
Add audio compression/limiting so quiet and loud radio traffic come out at a similar level.

Options, in order of preference:
1. Replace Darkice with Liquidsoap. Built-in `compress.rms`, `normalize`, limiting. One script handles capture, processing and the Icecast output.
2. Keep the stack, pipe capture through FFmpeg with the `acompressor` filter into Icecast.

Starting point for the FFmpeg route:

    ffmpeg -f alsa -i hw:1,0 \
      -af "acompressor=threshold=-20dB:ratio=4:attack=5:release=50" \
      -codec:a libmp3lame -b:a 128k \
      -f mp3 icecast://source:CHANGE_ME@127.0.0.1:8000/radio

Starting point for Liquidsoap (check syntax against the installed version):

    source = input.alsa(device="hw:1,0")
    source = compress.rms(source)
    source = normalize(source)
    output.icecast(%mp3(bitrate=128),
      host="127.0.0.1", port=8000, password="CHANGE_ME",
      mount="/radio", source)

## Other open items
- Reference the sound card by name instead of number (card index moved from 2 to 1 after a reboot). Check `arecord -L` for the `CARD=Pro` style name.
- Confirm UFW is limiting ports 8000 and 8080 to `tailscale0`.
- Disable Tailscale key expiry for this device in the admin console.
- Beeping on the stream when there is no input. Not diagnosed. Test with `arecord -D hw:1,0 -f cd -d 5 /tmp/test.wav && aplay /tmp/test.wav` to see if it is in the raw capture.
- Optional: recording (Darkice `[file-0]` section, or cron + streamripper).
- Optional: Whisper transcription + Claude summary pipeline.
