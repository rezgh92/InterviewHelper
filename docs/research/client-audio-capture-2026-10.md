# Client-side options: shell, audio capture, overlay panel (research notes, 2026-10-14)

Supporting research for `docs/ARCHITECTURE.md` section 4. Web-only research;
several vendor doc hosts were unreachable from the research sandbox, so some
specifics come from secondary sources and are marked "verify". Material on
hiding windows from screen capture was deliberately left out; see the ground
rules in `ARCHITECTURE.md`.

## 1. Bottom line

- Two-channel capture is the key design decision: mic = candidate,
  system/loopback = interviewer. Speaker attribution comes for free and only
  the interviewer channel needs turn detection.
- Best general-purpose stack today: **Electron 44/45 + Deepgram multichannel
  streaming**. Electron has built-in system-audio loopback on Windows (WASAPI)
  and macOS (Core Audio taps by default since Electron 39 / Chromium 142), plus
  `type: 'panel'` and always-on-top levels. Deepgram is the one major vendor
  that accepts a 2-channel stream on one WebSocket and returns per-channel
  transcripts (Nova-3 and Flux).
- Leaner alternative: **Tauri 2 (Rust)** with WASAPI loopback + Core Audio taps
  or ScreenCaptureKit done natively; Meetily and Hyprnote/Char are open-source
  Tauri apps that already do this capture.
- macOS-only premium path: Swift + Core Audio taps + SpeechAnalyzer (on-device,
  free, macOS 26) + NSPanel.
- Browser-only is possible in Chrome (system audio on macOS since Chrome 141;
  Document Picture-in-Picture gives an always-on-top panel) but the UX is poor:
  whole-screen share prompt every session, Chrome-only, no control over panel
  placement.
- The hard unsolved problem is not capture but **echo when the user is on
  speakers** (the interviewer's voice lands on both channels). Plan for a
  headphones-first UX plus AEC3 using the loopback as reference.

## 2. System-audio capture per OS

**macOS**
- Core Audio process taps (macOS 14.2+): `CATapDescription` →
  `AudioHardwareCreateProcessTap` → add tap to an aggregate device → read it
  like a microphone. Per-process selection, mono/stereo mixdown, optional mute
  of the source process, private taps. Requires `NSAudioCaptureUsageDescription`
  in Info.plist; the system prompts the first time; the app then appears in
  Privacy & Security > "Screen & System Audio Recording" as *System Audio
  Recording Only*. Gotchas: a missing plist key → calls succeed but buffers are
  all zeros (no error); launched from Terminal/IDE the permission is attributed
  to the parent app; no public API to query/request this permission. Apple
  sample: "Capturing system audio with Core Audio taps".
- ScreenCaptureKit audio (macOS 13+): `SCStreamConfiguration.capturesAudio`,
  `excludesCurrentProcessAudio`, per-app `SCContentFilter`. Needs the Screen
  Recording permission and, since Sequoia, a monthly re-approval prompt for
  apps not using `SCContentSharingPicker`. Overkill for audio-only.
- Chromium/Electron: Electron 39+ makes Core Audio taps the default loopback
  path on 14.2+; the opt-out flag was removed upstream in Chromium M155
  (Electron docs PR #53646, Sept 2026), so from ~Electron 46 it cannot be
  disabled. On macOS 13–14.1 Chromium uses the SCK path; on ≤12.7.6 no system
  audio without a virtual device.
- Chrome 141 (Sept 2025) added a "share system audio" toggle to
  `getDisplayMedia` on macOS (community-verified).
- Virtual devices (BlackHole, Loopback) remain the fallback for old macOS; poor
  UX.
- On-device STT: `SpeechAnalyzer` / `SpeechTranscriber` (macOS 26+), streaming
  via volatile results, models via `AssetInventory`, ~40 locales (verify with
  `SpeechTranscriber.supportedLocales`).

**Windows**
- WASAPI loopback (`AUDCLNT_STREAMFLAGS_LOOPBACK`): whole output mix, no prompt.
- Process loopback (`ActivateAudioInterfaceAsync` on `VAD\Process_Loopback`
  with `AUDIOCLIENT_PROCESS_LOOPBACK_PARAMS{TargetProcessId, INCLUDE/EXCLUDE}`):
  capture only the Zoom/Teams/browser process tree. Requires Windows 10 build
  20348+ (effectively Windows 11). Reference: Microsoft ApplicationLoopback
  sample, GStreamer wasapi2.
- Electron: `session.setDisplayMediaRequestHandler(..., { audio: 'loopback' |
  'loopbackWithMute' })` → whole system mix via `getDisplayMedia`.
- Libraries: NAudio `WasapiLoopbackCapture` (.NET), Rust `wasapi` crate,
  miniaudio, PyAudioWPatch (Python).

**Linux**
- PipeWire/PulseAudio monitor sources (`<sink>.monitor`), per-app capture via
  PipeWire links (≥0.3.50); no prompt. Chrome/Electron system audio through
  `getDisplayMedia` on Linux is unconfirmed, so do it natively
  (libpulse / pipewire-rs). Wayland always-on-top depends on the compositor.

## 3. Framework options

| Option | System audio | Mic | Overlay | Notes |
|---|---|---|---|---|
| **Electron** (44.4.5 stable 2026-09-23, Chromium 152; 45 due 2026-10-20) | Built-in: WASAPI (Win), Core Audio taps (macOS 14.2+), SCK (13–14.1) via `getDisplayMedia` + `setDisplayMediaRequestHandler` | `getUserMedia` (WebRTC AEC/NS, but AEC does not cancel other apps' playback) | `type: 'panel'` (non-activating NSPanel, floats over full-screen apps, all Spaces), `setAlwaysOnTop(true, 'screen-saver')`, `setVisibleOnAllWorkspaces(true, { visibleOnFullScreen: true })`, `focusable: false`, `setIgnoreMouseEvents` | Needs `NSAudioCaptureUsageDescription` + `NSMicrophoneUsageDescription`; when launched from a terminal/IDE that parent needs the key or you get a silent stream. Downsample in an AudioWorklet to 16 kHz PCM, interleave L/R, send over WebSocket. |
| **Tauri 2** | Native Rust: `wasapi` crate (Win); macOS via `screencapturekit` crate, `cidre` (used by Meetily), or Core Audio taps through `objc2` / `coreaudio-sys`; `tauri-plugin-system-audio` = WASAPI loopback + AEC3, mic-only on macOS | `cpal` | `alwaysOnTop`, `visibleOnAllWorkspaces`, `focus: false`, `transparent`, `decorations: false`; `tauri-nspanel` for NSPanel behaviors | WKWebView cannot do `getDisplayMedia`, so capture must be in Rust. Smaller binary, more work. References: Meetily (MIT), Hyprnote/Char, screenpipe. |
| **Swift native (macOS only)** | Core Audio taps (AudioCap reference app; AudioTee CLI pipes tap audio to stdout, usable as a sidecar) | AVAudioEngine, optional VPIO | NSPanel `.nonactivatingPanel`, `level = .floating / .screenSaver`, `collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]` | Add SpeechAnalyzer for free on-device STT; needs a separate Windows codebase. |
| **.NET (WinUI/WPF) + NAudio** | WASAPI / process loopback | NAudio | `WS_EX_TOPMOST | WS_EX_NOACTIVATE | WS_EX_TOOLWINDOW` | Windows-only unless Avalonia + native macOS interop. |
| **Flutter** | `audio_io` (WASAPI loopback + Core Audio taps), `desktop_audio_capture` (early), `liquid_speech` (SpeechAnalyzer, macOS 26 only) | `record` | `window_manager` | Plugin maturity risk. |
| **Python (prototype)** | PyAudioWPatch (Win); `catap` / `macloop` (macOS) | sounddevice | PyQt/Tk topmost | Good for benchmarking STT vendors on your own audio; hard to ship. |
| **Web / PWA / extension** | Chrome-only: `getDisplayMedia({ audio: true, systemAudio: 'include' })` whole-screen share → system audio on Windows/ChromeOS and macOS (Chrome 141+); tab audio in all Chromium browsers. Firefox ignores audio; Safari: no audio. | `getUserMedia` | Document Picture-in-Picture: Chrome/Edge 116+, Firefox 151, not Safari; not positionable by the page, dies with the opener tab, needs a user gesture | Prompt every session; cannot capture Zoom/Teams native apps from an extension. |

## 4. Transcription services (streaming), figures mid-2026, partly from third-party trackers

- **Deepgram**: Nova-3 streaming ~$0.0048/min (promo PAYG); **Flux** ~$0.0065/min
  English, Flux Multilingual GA 2026-04-29 (10 languages) ~$0.0078/min.
  `multichannel=true&channels=2` documented for Nova and Flux streaming → one
  socket, per-channel transcripts. Self-hosted/EU available. Best fit for the
  two-channel design.
- **AssemblyAI**: Universal-Streaming $0.15/hr; Universal-3.5/3.6 Pro Realtime
  $0.45/hr. Billed by session duration; multichannel streaming = one session per
  channel, so two channels ≈ 2x cost. Vendor-reported Pipecat benchmark:
  1.22% WER vs Flux 1.62% vs Scribe v2 3.12%.
- **OpenAI Realtime API** (`session.type: "transcription"`, WebSocket or WebRTC,
  server/semantic VAD): mono per session; no streaming diarization.
- **ElevenLabs Scribe v2 Realtime**: ~150 ms, 90+ languages, ~$0.39/hr.
- **Soniox** stt-rt: $0.12/hr flat incl. diarization/translation (vendor claim).
- **Speechmatics**: real-time diarization included, ~55 languages.
- **Gladia Solaria**: real-time, code-switching; $0.75/hr top plan.
- **Mistral Voxtral Realtime**: API ~$0.006/min; open weights
  `Voxtral-Mini-4B-Realtime-2602` (Apache 2.0, 13 languages; sub-200 ms
  configurable).

**On-device/local**: Apple SpeechAnalyzer (macOS 26); FluidAudio (Parakeet on
CoreML/ANE, streaming "Parakeet EOU 120M"); WhisperKit; whisper.cpp /
parakeet.cpp; Moonshine Voice (Feb 2026, MIT, true streaming); NVIDIA Nemotron
Speech Streaming (600M INT8 on CPU via sherpa-onnx); Kyutai STT 1B en/fr with
semantic VAD; Voxtral Mini Realtime.

**Skip-STT variant**: send audio straight to a multimodal realtime model and ask
for coaching text with input transcription as a side output. Less control,
hallucination risk, one hop fewer. Set aside in the architecture.

## 5. Echo / speaker-bleed mitigation

- Headphones-first UX; detect bleed (interviewer energy on both channels) and warn.
- AEC3 with the loopback as far-end reference: `tauri-plugin-system-audio`
  (Windows), Nonstrict RecordKit (commercial, macOS 14.2+, taps + AEC3, Electron
  bindings), Rust `aec3` crate / webrtc-audio-processing. Delay alignment is the
  hard part.
- macOS VPIO (`AVAudioEngine.inputNode.isVoiceProcessingEnabled`) cancels audio
  the engine itself plays; whether it cancels other apps' output is unverified.
- Cheap heuristic: drop mic-channel segments that duplicate recent
  system-channel text.

## 6. Server-side / bot alternatives

- Zoom **RTMS**: GA 2025-06-25; self-service purchase since May 2026; requires a
  Marketplace app + user install; Zoom only.
- Google **Meet Media API**: Developer Preview (restricted scopes; participants
  notified).
- Teams: application-hosted media bots only.
- **Recall.ai Desktop Recording SDK** (Mac/Win, botless, detects Zoom/Meet/Teams):
  $0.50/recording-hour + $0.15/hr transcription. A drop-in for the capture layer
  if we would rather buy it.

## 7. Recommended path and open items

1. Prototype with Electron 44/45: `getUserMedia` (mic) + `getDisplayMedia`
   loopback (system), AudioWorklet → 16 kHz stereo PCM → Deepgram Flux with
   `multichannel=true&channels=2`; panel = `BrowserWindow({ type: 'panel',
   alwaysOnTop: true, focusable: false, frame: false })`. Ship Info.plist keys;
   test from a packaged build, not a terminal.
2. If binary size/CPU matter later, port capture to Tauri/Rust using Meetily's
   audio module as the template.
3. Verify on real machines: Electron 45/46 tap behavior after M155; Linux
   loopback in Electron; AssemblyAI single-session dual-channel mode; VPIO
   cross-app AEC; all prices.

## Sources

- Electron: https://www.electronjs.org/docs/latest/api/desktop-capturer · https://releases.electronjs.org/pr/53646 · https://releases.electronjs.org/pr/49717 · https://www.electronjs.org/docs/latest/api/structures/base-window-options · https://github.com/alectrocute/electron-audio-loopback · https://endoflife.date/electron
- Chrome / web: https://blog.addpipe.com/getdisplaymedia-allows-capturing-the-screen-with-system-sounds-on-chrome-on-macos/ · https://addpipe.com/getdisplaymedia-demo/ · https://developer.chrome.com/docs/web-platform/document-picture-in-picture
- Apple: https://developer.apple.com/documentation/coreaudio/capturing-system-audio-with-core-audio-taps · https://developer.apple.com/documentation/bundleresources/information-property-list/nsaudiocaptureusagedescription · https://developer.apple.com/documentation/screencapturekit/scstreamconfiguration/capturesaudio · https://github.com/insidegui/AudioCap · https://github.com/makeusabrew/audiotee · https://www.recall.ai/blog/core-audio-taps · https://dev.to/nickdelv/2000-buffers-of-nothing-3i8 · https://9to5mac.com/2024/08/14/macos-sequoia-screen-recording-prompt-monthly/ · https://developer.apple.com/documentation/speech/speechanalyzer · https://developer.apple.com/documentation/speech/speechtranscriber · https://docs.fluidinference.com
- Windows: https://learn.microsoft.com/en-us/windows/win32/api/audioclientactivationparams/ns-audioclientactivationparams-audioclient_process_loopback_params · https://learn.microsoft.com/en-us/windows/win32/api/audioclientactivationparams/ne-audioclientactivationparams-audioclient_activation_type
- Tauri: https://docs.rs/tauri-plugin-system-audio · https://docs.rs/crate/screencapturekit/8.0.1/source/examples/README.md · https://docs.rs/crate/wasapi/latest · https://docsearch.algolia.com/mcp/docs/repo/ahkohd/tauri-nspanel · https://docsearch.algolia.com/mcp/docs/repo/zackriya-solutions/meetily · https://char.com/docs/faq
- Flutter / Python: https://pub.dev/documentation/audio_io/0.6.1/ · https://pub.dev/documentation/liquid_speech/latest/ · https://pypi.org/project/PyAudioWPatch/0.2.12.1 · https://pypi.org/project/catap/ · https://pypi.org/project/macloop/
- AEC: https://docs.rs/crate/aec3/0.1.8 · https://nonstrict.eu/recordkit/guides/echo-cancellation.html
- STT vendors: https://developers.deepgram.com/docs/multichannel · https://deepgram.com/learn/deepgram-launches-flux-multilingual-press-release · https://www.assemblyai.com/pricing.md · https://assemblyai.com/docs/streaming/multichannel-streams · https://www.assemblyai.com/blog/universal-3-5-pro-realtime · https://developers.openai.com/api/docs/guides/realtime-transcription · https://elevenlabs.io/realtime-speech-to-text-api · https://soniox.com/speech-to-text · https://arxiv.org/pdf/2602.11298 · https://docs.mistral.ai/studio-api/audio/overview
- Local models: https://petewarden.com/2026/02/13/announcing-moonshine-voice/ · https://arxiv.org/pdf/2604.14493 · https://kyutai.org/stt
- Bots / SDKs: https://developers.zoom.us/changelog/rtms/june-25-2025/ · https://developers.google.com/workspace/meet/docs/release-notes · https://www.recall.ai/product/desktop-recording-sdk · https://www.recall.ai/blog/new-recall-ai-pricing-for-2026
