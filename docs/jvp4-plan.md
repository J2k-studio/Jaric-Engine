# jvp4 — the video player, all in J2K  (owner, 2026-10-23: "the video player is Jvp4, with every function; all the programs we make ourselves, in J2K")

## The rule
No codec of Android (`MediaPlayer`, `MediaCodec`), no library of C. The container reader, the decoders, the sound mixer and the player are J2K. The app (Jaric OS) gives only two things: **a surface to put pixels on** (`7777;gfx`, done) and **a channel for PCM sound** (to make: `7777;pcm`, an `AudioTrack` fed from a file ring). Until each step is done, `jaric video` (Android's player) stays as the way to watch.

## The steps  (each ends with a check that runs on the phone)
| # | Step | Check |
|---|---|---|
| V1 | **mp4 / mov demuxer**: boxes, `moov`, tracks, sample tables (`stts stsc stsz stco ctts stss`), fragmented mp4 (`moof`) — **`jvp4 info` done (0.1.0)** | `jvp4 info` on real files; a list of the samples with their times |
| V2 | **PCM channel in the app** (`pcm;open;rate;channels;PATH`, `AudioTrack`) and the clock | `jvp4 beep` plays a sine made by J2K |
| V3 | **Motion JPEG** (baseline JPEG decoder: Huffman, IDCT, YCbCr to RGB) — also gives `<image>` for png/jpeg in the engine | a MJPEG avi / mov plays on the surface |
| V4 | **H.264 / AVC decoder**: NAL units, SPS/PPS, CAVLC (baseline, main), intra and inter prediction, deblocking; later CABAC (main, high) | a baseline mp4 plays; bit-exact against a reference on test streams |
| V5 | **AAC-LC decoder** (and MP3, Opus later): MDCT, Huffman, TNS off at first | the sound of an mp4 plays in time with the picture |
| V6 | **Player**: A/V sync by the sound clock, pause, seek (nearest key frame), speed, subtitles (`srt`, `vtt`, mp4 `tx3g`), volume, loop, playlist, fullscreen, keys and touch | `jvp4 play FILE` |
| V7 | **More containers and codecs**: mkv / webm (EBML), avi, ts; VP8, VP9, AV1 (later; AV1 and HEVC are big); FLAC, Vorbis | each with a test file |
| V8 | **Speed**: SIMD (`vec`/NEON) in IDCT, motion compensation and colour conversion; `#multithread` for slices / frames; the GPU does the colour conversion and the scaling (`<gpu>`: the decoder writes YUV planes, a shader makes RGB) | 1080p30 H.264 on this phone |

## How big (honest numbers, J2K lines)
mp4 demuxer 600 · JPEG 900 · H.264 4000 to 6000 (CAVLC + CABAC + inter + deblock) · AAC-LC 2000 · player and sync 800 · mkv 700. Several long sessions; V1 to V3 are small, V4 is the mountain.

## What the engine gets from it
The JPEG and PNG decoders (V3, and PNG with `<deflate>` which exists) become the picture loader of Jaric Engine (sprites from files without the OS); the PCM channel (V2) becomes real sound for games (many voices, mixing in J2K).

## Related: `tree` (owner, same message)
`tree` of jbox shows the folders (`tree -d`, `tree -s`, `-L N`); for the root: `tree -L 2 /` is not allowed by Android in Termux and in the app (a folder that cannot be read is shown as `[cannot read this folder]`); the root of Jaric OS (the folders of the app: `usr`, `home`, `tmp`) is `tree $PREFIX/..`. When the OS has its own file system the real `:/` can be listed.
