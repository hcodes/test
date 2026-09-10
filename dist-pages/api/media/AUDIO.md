# Demo audio

`both.sbc` is derived from the local checkdevice project's
`static/sound/headphones/both.mp3`. The full short sample is included.

`both-low.sbc` is an experimental option: the same 32 kHz source with bitpool 24,
61-byte SBC frames, 122 kbit/s, 1,320 frames, 5.280 seconds, 80,520 bytes.
The original `both.sbc` is available at bitpool 48,
109-byte frames and 218 kbit/s. Lower bitrate is independent of sample rate.
The player preserves report 0x17 for four frames at either bitrate. A smaller
0x14 report carrying four 61-byte frames produced no sound on hardware and was
reverted. The lower-bitrate codec still needs validation with 0x17; this framing
keeps the same Bluetooth report size despite the smaller encoded file.

The previous default remains selectable as `both-16k-low.sbc`: 16 kHz, bitpool 24, 61 kbit/s,
61 bytes per frame, 664 frames, 5.312 seconds, 40,504 bytes. The complete source
is retained; codec-flush silence and packet padding account for the small duration
difference. Each frame now lasts 8 ms, so four-frame reports are paced at 32 ms
and the player drains the final buffer at the same rate. All three files remain
selectable. Hardware acceptance of the 16 kHz option is not yet confirmed.

Conversion downmixes the channels to mono, normalizes PCM peak to 12%, applies
16 ms fades, duplicates the result into both channels, and adds trailing silence
to flush the codec and finish a four-frame HID packet.

- Codec: SBC, 32 kHz, joint stereo, SNR allocation, 8 subbands, 16 blocks, bitpool 48.
- Frame size: 109 bytes / 4 ms.
- Output: 1,320 frames, 5.280 seconds, 143,880 bytes.
- Encoding: BlueZ `sbcenc` 2.1; MP3 decoding and resampling: macOS `afconvert`.

From the repository root on macOS, with `sbcenc` available:

```sh
python3 scripts/convert-speaker-audio.py \
  ../checkdevice/static/sound/headphones/both.mp3 \
  demo/both.sbc --sbcenc /path/to/sbcenc

python3 scripts/convert-speaker-audio.py \
  ../checkdevice/static/sound/headphones/both.mp3 \
  demo/both-low.sbc --sbcenc /path/to/sbcenc --bitpool 24

python3 scripts/convert-speaker-audio.py \
  ../checkdevice/static/sound/headphones/both.mp3 \
  demo/both-16k-low.sbc --sbcenc /path/to/sbcenc --bitpool 24 --sample-rate 16000
```

The browser downloads the encoded asset through Parcel's `url:` import handling.
No MP3 decoder, SBC encoder, or WebAssembly module runs in the demo.

The demo's build target includes Chrome 104 so Parcel uses its URL runtime instead
of `import.meta.resolve()`. With Parcel 2.16.4 and a latest-Chrome-only target, the
SBC URL referenced a missing import-map entry and prevented the demo from loading.

## Native macOS comparison

Close the browser demo and connect exactly one Sony DualShock 4 over Bluetooth.
The diagnostic uses Swift and IOKit, without WebHID or runtime audio encoding.
Node dependencies (`npm install`) and Xcode Command Line Tools (`xcode-select --install`,
if missing) are required. Run these commands from the repository root:

```sh
npm run speaker:native -- --list
npm run speaker:native
npm run speaker:native -- --tone
```

The default plays `demo/both-16k-low.sbc` at 16 kHz / 61 kbit/s, four frames per
report, input interval 8, and send-ahead 256 ms. `--tone` plays the exact same
32 kHz / 800 ms signal as the browser's Test speaker. The utility sets the
lightbar dark blue, stops rumble, and attempts to mute the speaker on completion,
write failure, or Ctrl+C. A blocking OS write must return before Ctrl+C cleanup
can finish. Disconnecting the device can prevent the mute command from succeeding.

Other comparisons and an offline validation:

```sh
npm run speaker:native -- --frames 1 --buffer 64
npm run speaker:native -- --file demo/both.sbc --poll 8 --buffer 256
npm run speaker:native -- --validate
```

`--list` only enumerates devices. `--validate` builds and checks the packet plan
without opening HID. The Swift executable is compiled in a temporary directory
and removed after each invocation. SBC validation, packet layout, and CRC generation
reuse the TypeScript implementation; IOKit receives the report ID followed by the
WebHID payload, with the synthetic CRC prefix omitted.

After playback, stdout contains JSON diagnostics. Compare `averageWriteMs`,
`maxWriteMs`, `averageGapMs`, and `elapsedMs` with the browser using the same file
and settings. These measure host API completion, not actual radio delivery or
controller FIFO depth. If native writes still average about 71 ms for 32 ms of
audio, bypassing the browser alone has not removed the bottleneck. If the timings
improve, listening on the physical controller is still necessary to confirm
continuous playback. A native run is an experiment, not a guaranteed stutter fix.


## Stronger compression experiment

The browser demo retains `both-16k-bp12.sbc`: 16 kHz, bitpool 12,
37 kbit/s, 664 frames, 5.312 seconds, 24,568 bytes. The selector also offers
`both-16k-bp8.sbc`: bitpool 8, 29 kbit/s, 19,256 bytes, with the same duration.
These use the same source, normalization, fades and channel mode. HID IDs,
packet sizes and frames per report remain unchanged. Native decoding and
packet tests do not confirm that a physical DS4 accepts these bitpools.
The Test speaker button still plays the original reference tone.

Regenerate with the conversion command above using `--sample-rate 16000`
and `--bitpool 12` or `--bitpool 8`. For native playback, select the file:

```sh
npm run speaker:native -- --file demo/both-16k-bp12.sbc
npm run speaker:native -- --file demo/both-16k-bp8.sbc
```

The native tool's default remains the 16 kHz / bitpool 24 baseline.


The browser default is now `both-16k-bp4.sbc`: 16 kHz / bitpool 4 / 21 kbit/s,
13,944 bytes. `both-16k-bp2.sbc` is selectable at bitpool 2 / 17 kbit/s,
11,288 bytes. Both contain 664 frames / 5.312 seconds. Native playback accepts
these files via `--file`; native defaults remain unchanged. Stronger compression
still preserves the HID packet sizes and four-frame default. Hardware decoding
of these experimental bitpools is not yet confirmed.


## Dense packet experiment result

The user reported silence whenever more than four frames were packed into the
current 0x17 layout. The browser now sends four frames per sample packet again;
the dense-packing controls have been removed. The existing **Compare packet sizes**
still uses 1/2/4 frames. Bitpool 2/4/8/12 produced sound, but did not resolve stutter.

The library/native 8/16/24-frame options are retained only for explicit protocol
diagnostics. Their parser, CRC, padding and pacing tests do not verify hardware
acceptance. This failure concerns the tested layout, not every possible DS4 mode.


## Selecting the baseline packet size

The **Frames per packet** selector offers 4 (0x17), 2 (0x14), and 1 (0x12),
with 4 selected initially. It applies to both **Test speaker** and **Play sample**.
Changing it leaves the selected sample profile, send-ahead and input interval
unchanged. **Compare packet sizes** still tests all three sizes automatically.
The selector is disabled during loading/playback. Counts above four remain
unavailable in the demo after the hardware test produced silence.
