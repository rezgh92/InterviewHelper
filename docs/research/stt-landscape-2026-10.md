# Real-time / streaming STT landscape (research notes, 2026-10-14)

Supporting research for `docs/ARCHITECTURE.md` section 5. Figures were seen on
2026-10-14 via web search excerpts and third-party benchmarks; direct vendor
pages were partly unreachable from the research sandbox. Items marked
**UNVERIFIED** conflicted across sources or could not be confirmed. Treat every
number as an input to the Phase 0 benchmark, not as a conclusion.

## 1. Cloud streaming STT APIs

| Vendor / model | Latency (reported) | Native endpointing / EOT | Streaming diarization | Word timestamps | Price (USD) | Protocol / audio | Free tier |
|---|---|---|---|---|---|---|---|
| **Deepgram Flux** (`flux-general-en`, launched 2025-10-02; Flux Multilingual GA 2026-04-29, 10 langs) | EOT median <300 ms, ~260 ms claimed, P90 ~1 s (vendor, cited by Coval); Coval TTFT ~1,086 ms | Yes, model-native: `eot_threshold` default 0.7 (0.5–1.0), `eot_timeout_ms` default 5000, opt-in `eager_eot_threshold` (0.3–0.9) emits EagerEndOfTurn / TurnResumed / EndOfTurn | **UNVERIFIED**: diarization doc lists Flux, but self-hosted Flux is "not compatible with diarizer" and Pipecat's Flux service exposes no `diarize`. Nova-3 streaming diarization works | Default on | Flux EN $0.0077/min list ($0.46/hr), $0.0065 promo; Flux Multi $0.0078/min; Nova-3 streaming $0.0077/min | WebSocket; 20–100 ms buffers per docs; `multichannel=true&channels=2` supported | $200 credit, no card; expiry disputed |
| **AssemblyAI Universal-3.6 Pro Realtime** (released 2026-09-29) / Universal-Streaming | ~150 ms P50 to final (vendor); Coval 138 ms time-to-final, 2.2% WER (reported by partner); Coval TTFT ~1,098 ms | Yes, acoustic+semantic in-model; 3.6 Pro defaults `min_turn_silence` 100 ms, `max_turn_silence` 1000 ms (Universal-Streaming: 400 / 1280 ms) | Yes: `speaker_labels: true`, `max_speakers` 1–10, end-of-session `SpeakerRevision` | Yes, per final word incl. speaker | 3.6 Pro $0.45/hr + $0.12/hr diarization; Universal-Streaming $0.15/hr + $0.04/hr; billed on socket time incl. idle, auto-close 3 h | WebSocket; 16 kHz pcm_s16le, 50 ms frames, allowed 50–1000 ms; one session per channel | $50 credits |
| **OpenAI** gpt-live-transcribe / gpt-realtime-whisper | Coval gpt-4o-transcribe mean 779 ms time-to-final (2026-10-02); first chunk 500–1,500 ms via Realtime (single tester). No official figure | `server_vad` or `semantic_vad` with eagerness low/medium/high | No; `gpt-4o-transcribe-diarize` is file-only | **UNVERIFIED** for realtime | Live transcription $0.017/min ($1.02/hr); gpt-4o-transcribe ≈$0.006/min batch | WebSocket or WebRTC; pcm16 24 kHz mono | None found |
| **Google Chirp 3** (Speech V2 `StreamingRecognize`) | Not published | Voice-activity events; speech begin/end timeouts 500 ms–60 s | **No**, BatchRecognize-only | Streaming **UNVERIFIED** | $0.016/min ($0.96/hr) | gRPC bidi stream; session ≤5 min, ≤25 KB/chunk | 60 min/month **UNVERIFIED for V2**; $300 new-customer credit |
| **Azure Speech** (ConversationTranscriber) | Not published; new English VAD (Jul 2025) cut ~100 ms | Silence segmentation 100–5000 ms plus optional semantic segmentation | Yes | Yes (SDK) | $1/hr real-time + $0.30/hr diarization; fast $0.36/hr; batch $0.18/hr | Speech SDK (WebSocket underneath) | F0: 5 h/month |
| **Speechmatics** | EOS→final 0.451 ± 0.022 s with Voice SDK turn detection; ForceEndOfUtterance→final ~250 ms | `max_delay` 0.7–4 s + `end_of_utterance_silence_trigger` | Yes; EndOfUtterance carries no speaker | Yes | Standard $0.24/hr; Enhanced $0.43/hr (cut 2026-07-06) | WebSocket | 50 h/month |
| **Gladia Solaria-1** | ~103 ms partial, ~270 ms final (vendor) | Silence-based `endpointing` + `maximum_duration_without_endpointing` | **UNVERIFIED / likely no** | Yes | $0.75/hr PAYG, ≈$0.25/hr Growth | WebSocket `/v2/live`; session ≤3 h | 10 h/month |
| **ElevenLabs Scribe v2 Realtime** | ~150 ms partial (vendor); Coval TTFT ~2,160 ms, ~123 ms time-to-final, ~4.47% pooled WER | VAD commit or manual commit | **No** | Yes | $0.39/hr; $0.28/hr annual Business | WebSocket; PCM or mu-law | ≈30 min/month **UNVERIFIED** |

Notes: Coval's "TTFT" (~1–2 s for every vendor) is a different metric from
end-of-speech→final and should not be read as pipeline lag. AssemblyAI's
vendor-run benchmark page reports Coval pooled WER of 2.30% (3.6 Pro), 4.47%
(Scribe v2 RT), 5.77% (Flux). A third-party summary showing Nova-3 at ~25% WER
looks anomalous and is **UNVERIFIED**.

## 2. On-device / open-source streaming

- **whisper.cpp / faster-whisper.** Whisper is a 30 s-window batch model;
  `whisper-stream` approximates streaming with `--step 500 --length 5000` or a
  VAD-gated sliding window. Practical lag is 0.5–2 s behind speech.
  large-v3-turbo runs ~17.7× real time on an M3 Ultra (batch). SimulStreaming
  (ufal, AlignAtt policy) claims ~5× faster than LocalAgreement, targeting ~2 s.
- **NVIDIA.** Parakeet-TDT-0.6B-v3 and Canary-1B-v2 are batch-oriented and
  degrade in streaming. **Nemotron Speech Streaming 0.6B** is cache-aware
  FastConformer-RNNT with runtime-switchable 80/160/560/1,120 ms chunks, ~24 ms
  median latency, only 0.21% abs WER loss vs batch; 2.47 GB, quantizable to
  0.67 GB; a CoreML port exists. Needs a GPU/ANE for comfort (CPU RTFx ≈2.5).
- **Moonshine v2 streaming** (paper Feb 2026): 80 ms algorithmic lookahead,
  built for 0.1–1 TOPS / <1 GB; TTFT ≈34 ms (MacBook Pro), 69 ms (x86), 237 ms
  (Raspberry Pi 5) for Tiny; English-only; can hallucinate on short/noisy audio.
- **Kyutai STT** (CC-BY-4.0): `stt-1b-en_fr` with 0.5 s delay and built-in
  semantic VAD; `stt-2.6b-en` 2.5 s delay; server-side flush trick cuts the wait
  to ~125 ms.
- **Apple SpeechAnalyzer** (iOS/macOS 26): on-device, volatile vs finalized
  results, per-word `audioTimeRange`, no speaker labels. Third-party benchmark:
  2.12% WER clean vs Whisper Small 3.74%, ~3× faster on M2 Pro.
- **Windows AI Speech Recognition API**: public preview (Build 2026); Windows
  App SDK 2.2.2-experimental9, Windows 11 24H2, MSIX; NPU on Copilot+ PCs, CPU
  fallback; English only.

## 3. VAD and turn-taking

- **Silero VAD v6**: ROC-AUC 0.97 vs WebRTC VAD 0.73; 512-sample (32 ms)
  chunks at 16 kHz, ~1 ms per chunk, ~1.2–1.8 MB. At 5% FPR, Silero 87.7% TPR
  vs WebRTC 50%.
- **Smart Turn v3** (Pipecat, BSD-2): Whisper-Tiny backbone ~8M params, 8 MB
  int8, 12 ms on modern CPU, 23 languages.
- **LiveKit Turn Detector v1** (2026-06-17): audio-native, 14 languages; v1
  hosted, v1-mini open-weight CPU model. LiveKit's EOT-Bench: v1 9.9% false
  cut-offs at a 300 ms budget vs Smart Turn v3.2 35.2%, v1-mini 27.8%. An
  independent paper found Smart Turn 13.5% false cut-off / 20.7% EOT recall vs
  LiveKit 2.7% / 91.5%. Evidence is vendor-heavy; test on your own audio.
- **Vendor-native EOT** (Flux, AssemblyAI, OpenAI semantic_vad, Speechmatics,
  Kyutai) is in-model and avoids a second network hop.
- **Deciding "the interviewer has finished asking."** Capture system audio
  (interviewer) and mic (candidate) as separate streams so attribution is by
  channel; run EOT only on the interviewer channel. Best: Flux EagerEndOfTurn at
  `eager_eot_threshold` ≈0.5–0.6 to prefetch, confirm on EndOfTurn, cancel on
  TurnResumed; or AssemblyAI 3.6 Pro with default 100/1000 ms silences. Local
  fallback: Silero VAD + ~700–900 ms hangover, then Smart Turn v3 on the last
  few seconds; a cheap text heuristic (trailing "?" / wh-word) reduces cuts on
  multi-sentence questions.

## 4. Orchestration frameworks

- **LiveKit Agents** requires a LiveKit server and room model; strong
  SIP/WebRTC.
- **Pipecat** (Python, BSD-2) is transport-agnostic; `LocalAudioTransport` runs
  on a laptop with no server; bundles Smart Turn.
- **Vocode** is telephony-focused with less momentum.
- **Verdict for a single-user desktop app:** call the STT WebSocket directly.
  Borrow Smart Turn v3 / v1-mini weights and Pipecat's provider adapters as
  libraries; adopt LiveKit only if a browser WebRTC client with server-side
  agents is needed later.

## 5. Recommendation

- **Default cloud:** Deepgram Flux (English) for native turn events, eager EOT
  and single-socket multichannel, ≈$0.46/hr. **Alternative:** AssemblyAI
  Universal-3.6 Pro Realtime for best WER (2.2–2.3%) or streaming diarization,
  $0.45 + $0.12/hr; mind idle-time billing and per-channel sessions.
- **Privacy-first on-device:** macOS → Apple SpeechAnalyzer + Silero VAD + Smart
  Turn v3; cross-platform GPU → Nemotron Speech Streaming 0.6B (160 ms chunks);
  CPU-only → Moonshine v2 streaming (English) or whisper.cpp large-v3-turbo with
  VAD chunking (0.5–2 s lag); Windows → Windows AI Speech Recognition API
  (preview, English).
- **Pipeline shape:** two 16 kHz mono PCM16 streams (24 kHz only for OpenAI);
  20–50 ms frames; local Silero VAD to gate uploads and mute; one STT socket
  per channel or one multichannel socket; show partials in the UI only; feed
  finals plus EndOfTurn to the LLM; prefetch on EagerEndOfTurn; rotate Google
  sessions every 5 min if used.

## Open items to verify in Phase 0

(a) whether hosted `flux-general-en` accepts `diarize`; (b) Google V2 free tier
and chirp_3 streaming word offsets; (c) Gladia live diarization status;
(d) OpenAI realtime word-timestamp support; (e) Deepgram $200 credit expiry;
(f) LiveKit v1-mini model size/latency. No vendor publishes Azure or Google
first-partial latency; measure them if they stay in scope.

## Sources (all seen 2026-10-14)

- Deepgram: https://developers.deepgram.com/docs/flux/configuration · https://developers.deepgram.com/reference/speech-to-text/listen-flux · https://developers.deepgram.com/docs/multichannel · https://developers.deepgram.com/docs/diarization · https://developers.deepgram.com/docs/flux-self-hosted · https://developers.deepgram.com/docs/measuring-streaming-latency · https://deepgram.com/learn/deepgram-launches-flux-multilingual-press-release · https://www.cartesia.ai/learn/deepgram-pricing
- AssemblyAI: https://www.assemblyai.com/pricing · https://www.assemblyai.com/blog/universal-3-6-pro-realtime · https://www.assemblyai.com/docs/streaming/label-speakers-and-separate-channels · https://www.assemblyai.com/docs/streaming/universal-streaming/turn-detection · https://www.assemblyai.com/docs/streaming/message-sequence · https://www.assemblyai.com/benchmarks · https://assemblyai.com/docs/streaming/multichannel-streams
- OpenAI: https://developers.openai.com/api/docs/pricing · https://developers.openai.com/api/docs/guides/realtime-vad.md · https://platform.openai.com/docs/guides/realtime-transcription · https://cookbook.openai.com/examples/speech_transcription_methods · https://developers.openai.com/api/docs/guides/speech-to-text
- Google: https://docs.cloud.google.com/speech-to-text/docs/models/chirp-3 · https://github.com/livekit/agents/issues/7684 · https://convertaudiototext.com/blog/google-cloud-speech-to-text-pricing-2026
- Azure: https://learn.microsoft.com/en-us/azure/ai-services/speech-service/releasenotes · https://docs.azure.cn/en-us/ai-services/speech-service/how-to-recognize-speech · https://learn.microsoft.com/en-us/azure/ai-services/speech-service/get-started-stt-diarization
- Speechmatics: https://www.speechmatics.com/pricing · https://docs.speechmatics.com/api-ref/realtime-transcription-websocket · https://docs.speechmatics.com/speech-to-text/realtime/turn-detection · https://www.speechmatics.com/company/articles-and-news/speed-you-can-trust-the-stt-metrics-that-matter-for-voice-agents
- Gladia: https://www.gladia.io/product/real-time · https://docs.gladia.io/chapters/live-stt/features/endpointing · https://docs.gladia.io/api-reference/v2/live/init
- ElevenLabs: https://elevenlabs.io/realtime-speech-to-text · https://elevenlabs.io/blog/real-time-speech-to-text-under-200ms · https://github.com/elevenlabs/skills/blob/main/speech-to-text/SKILL.md
- Benchmarks: https://benchmarks.coval.ai/stt · https://www.coval.ai/blog/best-speech-to-text-providers-in-2026-independent-benchmarks-and-how-to-choose/ · https://livekit.com/benchmarks/eot-bench · https://arxiv.org/pdf/2603.13379
- On-device: https://github.com/ggml-org/whisper.cpp/blob/master/examples/stream/README.md · https://github.com/ufal/SimulStreaming · https://arxiv.org/html/2604.14493v2 · https://huggingface.co/blog/nvidia/nemotron-speech-asr-scaling-voice-agents · https://huggingface.co/FluidInference/nemotron-speech-streaming-en-0.6b-coreml · https://www.alphaxiv.org/abs/2602.12241 · https://huggingface.co/UsefulSensors/moonshine-streaming-tiny · https://github.com/kyutai-labs/delayed-streams-modeling/ · https://developer.apple.com/videos/play/wwdc2025/277/ · https://get-inscribe.com/blog/apple-speech-api-benchmark.html · https://learn.microsoft.com/en-us/windows/ai/apis/speech-recognition
- VAD / turn-taking: https://github.com/snakers4/silero-vad/wiki/Quality-Metrics · https://picovoice.ai/blog/best-voice-activity-detection-vad/ · https://www.daily.co/blog/announcing-smart-turn-v3-with-cpu-inference-in-just-12ms/ · https://github.com/pipecat-ai/smart-turn · https://livekit.com/blog/solving-end-of-turn-detection · https://docs.livekit.io/agents/logic/turns/turn-detector/
- Frameworks: https://www.forasoft.com/blog/article/pipecat-vs-livekit-agents · https://github.com/pipecat-ai/pipecat/blob/main/examples/README.md
