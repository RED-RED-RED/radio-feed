# TODO

## Compression (next up)
Add audio compression/limiting so quiet and loud radio traffic come out at a similar level.

Options, in order of preference:
1. Replace Darkice with Liquidsoap. Built-in `compress.rms`, `normalize`, limiting. One script handles capture, processing and the Icecast output.
2. Keep the stack, pipe capture through FFmpeg with the `acompressor` filter into Icecast.

Starting point for the FFmpeg route:

    ffmpeg -f alsa -i hw:CARD=Device,DEV=0 \
      -af "acompressor=threshold=-20dB:ratio=4:attack=5:release=50" \
      -codec:a libmp3lame -b:a 128k \
      -f mp3 icecast://source:CHANGE_ME@127.0.0.1:8000/radio

Starting point for Liquidsoap (check syntax against the installed version):

    source = input.alsa(device="hw:CARD=Device,DEV=0")
    source = compress.rms(source)
    source = normalize(source)
    output.icecast(%mp3(bitrate=128),
      host="127.0.0.1", port=8000, password="CHANGE_ME",
      mount="/radio", source)

## Other open items
- Verify Darkice runs cleanly as the non-root `darkice` user with `realtime = no`.
- Confirm UFW is limiting ports 8000 and 8080 to `tailscale0`.
- Disable Tailscale key expiry for the device in the admin console.
- Occasional beeping on the stream when there is no input. Not diagnosed. Record straight from the card with `arecord` to see whether it is in the raw capture.
- Optional: recording (Darkice `[file-0]` section, or cron + streamripper).
- Optional: Whisper transcription + LLM summary pipeline.
