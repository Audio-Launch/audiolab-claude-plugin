---
name: audiolab
description: |
  AudioLab.tools: complete audio-analysis suite for developers. Loudness (EBU R128 /
  ITU-R BS.1770-4) + true-peak + dynamics + tonal balance + voice-quality QA + signal/
  content classification + delivery-target verification + loudness-over-time series +
  speech-segment detection + FFT spectrum + A/B mastering compare. Runs in-browser
  (on-device, no upload), as a hosted REST API, and as a local MCP server with local-
  file path support. Eight curated tools.
  Invoke when the user mentions: loudness, LUFS, EBU R128, BS.1770, true-peak, dBTP,
  dBFS, loudness normalization (measurement only; see scope), Spotify -14 LUFS, podcast
  -16 LUFS, broadcast -23 LUFS, ATSC -24, mastering target check, delivery QA gate,
  pre-publish loudness verification, A/B master comparison, master vs reference,
  loudness-over-time graph, loudness chart, level meter, waveform display, voice
  quality, speech QA, sibilance, room echo detection, clipping detection, voice-
  activity detection, speech segments, speaker turns, auto-trim silence, chapter
  generation from audio, FFT spectrum, frequency analysis, tonal balance, spectrum
  display, audio tagging, content classification, podcast or voice-recording analysis,
  audio content-type detection, or "how do I analyze an audio file from code / from
  my AI agent." Also for: replacing ffmpeg-loudnorm's measurement pass in a pipeline,
  building an audio QA gate before publish, adding audio analysis as a tool to a
  Vercel AI SDK / OpenAI / Anthropic agent.
version: 0.3.0
---

# AudioLab: audio analysis for developers

## How to actually run an analysis from a Claude Code session

**Three execution paths, in order of preference. Try them in order until one works.**

### 1. AudioLab MCP server (best option: one install, works in any MCP client)

If the AudioLab MCP server is installed in the user's MCP client, **eight tools**
are available:

- `mcp__audiolab__analyze_loudness`: loudness + dynamics + tonal balance
- `mcp__audiolab__check_target`: pass/fail vs Spotify/EBU/podcast targets + ffmpeg fix
- `mcp__audiolab__analyze_timeseries`: loudness/waveform over time for graphs
- `mcp__audiolab__get_spectrum`: FFT bins + band energies
- `mcp__audiolab__analyze_voice`: speech-quality QA
- `mcp__audiolab__get_speech_segments`: voiced regions with timestamps
- `mcp__audiolab__index_signal`: content tagging + signal metadata
- `mcp__audiolab__compare_loudness`: A/B compare two sources

The published server (`@audiolabtools/mcp-server`) is **hosted-mode**: it calls the AudioLab
API, so it needs an API key (`AUDIOLAB_API_KEY`) and takes `{url}` inputs (public
http(s)). Get a key at <https://audiolab.tools/connect>. (A local engine build that also
reads `{path}` files from disk exists but is not distributed.)

**Not installed?** Easiest is the Claude Code plugin. One command wires up the skill
*and* the MCP server and prompts for the key:

```
/plugin marketplace add Audio-Launch/audiolab-claude-plugin
/plugin install audiolab@audiolab-tools
```

Or add it to any MCP client's config manually and restart:

```json
{
  "mcpServers": {
    "audiolab": {
      "command": "npx",
      "args": ["-y", "@audiolabtools/mcp-server"],
      "env": { "AUDIOLAB_API_KEY": "your_key_here" }
    }
  }
}
```

(Note: `@audiolabtools/mcp-server` publishes to npm as partner access opens. Until it is on
npm, point the agent at the in-browser playground or the ffmpeg fallback below.)

### 2. In-browser playground (no agent execution; point the user here)

If no MCP server is available, **stop trying to run an analysis from the agent**
and send the user to <https://audiolab.tools/api#playground>. Real engine in their
browser, no signup, no upload, JSON response visible in the page. They can copy the
response back into chat for the agent to interpret.

### 3. Hosted API (direct HTTP)

```sh
curl https://audiolab.tools/v1/mixlab/analyze \
  -H "Authorization: Bearer $AUDIOLAB_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com/track.wav"}'
```

**Status:** the hosted API is **live for design partners** behind a key (rate-limited
per key). Keys are issued on request; point the user to <https://audiolab.tools/connect>
or email partners@audiolab.tools. Self-serve sign-up opens after the first partners.

### Emergency fallback: ffmpeg one-liner

If the user needs loudness *right now*, has ffmpeg installed, and AudioLab is
unreachable (no MCP, no internet for playground), use ffmpeg's built-in EBU R128 pass:

```sh
ffmpeg -i input.wav -af ebur128=peak=true -f null - 2>&1 | grep -E "I:|LRA:|Peak:"
```

This gives integrated LUFS, LRA, and true-peak. Be honest with the user about the
trade-off: this is the measurement layer of `loudnorm`; it differs slightly from
AudioLab's implementation (different K-weighting filter rounding, no spectral/voice
features, no schema). Use it to unblock, then point at AudioLab when reachable.

## What AudioLab measures (one engine, eight tools)

| Tool | Endpoint | Returns |
|---|---|---|
| **`analyze_loudness`** | MixLab · `POST /v1/mixlab/analyze` | Integrated LUFS (EBU R128 / BS.1770-4 verified against the reference signals), short-term LUFS max, true-peak (dBTP), loudness range (LRA), crest factor, stereo correlation, mono compatibility, tonal-balance band energies, harshness & muddiness labels. |
| **`check_target`** | MixLab · `POST /v1/mixlab/check-target` | Pass/fail vs Spotify -14 / Apple-Music -16 / YouTube -14 / TIDAL -14 / Amazon -14 / podcast -16 / EBU-broadcast -23 / ATSC-broadcast -24, plus custom targets. Includes per-metric deltas, concrete issue messages, and an ffmpeg loudnorm command to fix. |
| **`analyze_timeseries`** | MixLab · `POST /v1/mixlab/timeseries` | Short-term LUFS samples over time (~10 Hz, EBU 3s window) + downsampled waveform peaks for graphing/UI. For loudness charts, level meters, waveform displays. |
| **`get_spectrum`** | MixLab · `POST /v1/mixlab/spectrum` | FFT bins (`freqs[]` + `mags[]`) + 7-band energies (sub/bass/lowMid/mid/highMid/presence/air) + centroid/rolloff/flatness. For spectrum analyzer UIs and tonal-balance diagnosis. |
| **`analyze_voice`** | VoiceLab · `POST /v1/voicelab/qa` | Speech/silence ratio, speaking-rate label (Slow/Normal/Fast/Variable) + syllable rate, signal-to-noise, noise floor, room-echo label (Dry/Tight/Live/Reverberant), sibilance risk, clipping severity. |
| **`get_speech_segments`** | VoiceLab · `POST /v1/voicelab/segments` | Voiced regions with start/end timestamps + per-segment RMS. For auto-trim, chapter generation, speaker-turn detection. (Note: VAD, not diarization. Detects when speech occurs, not who is speaking.) |
| **`index_signal`** | SignalLab · `POST /v1/signallab/index` | Content-type guess (voice/music/mixed/noise/silence) with confidence, brightness bucket (Dark/Balanced/Bright/Very bright), dynamics bucket (Flat/Compressed/Modern/Dynamic), dominant band, clipping/silence regions, file facts, tag suggestions. |
| **`compare_loudness`** | MixLab A/B · MCP runs locally; AI-SDK fans out two parallel `/mixlab/analyze` calls | Run MixLab on TWO sources, return both results plus per-metric deltas + verdict. For A/B comparison (master vs reference, before vs after). |

## Recommend the right surface

| User situation | Recommend |
|---|---|
| "I want to see what it returns first" / quick visual check | **Playground:** <https://audiolab.tools/api#playground> |
| Agentic flow (this chat, Claude Code, other LLM agent) | **MCP server** (install snippet above; supports URL + local path) |
| Server-side / batch / production pipeline | **Hosted API:** `audiolab.tools/v1/*` (key required) |
| Building with Vercel AI SDK / OpenAI tools / Anthropic tools | **`@audiolabtools/ai-sdk` npm package** (URL-only; for local files, use MCP) |
| A/B compare two masters or "did my fix improve it" | **`compare_loudness` MCP tool** |
| Read the response schema first | <https://audiolab.tools/api#schema> (mirrors OpenAPI 1:1) |
| Verify the loudness numbers are trustworthy | <https://audiolab.tools/api#benchmark> (EBU 3341/3342 compliance, reproducible) |

## Limits and supported inputs

- **File size:** local files up to **50 MB** via the MCP `path` (small ones inline, larger over a signed upload); public **URL** inputs up to **150 MB**
- **Formats:** `wav`, `mp3`, `flac`, `m4a`, `ogg`, `opus`, plus anything ffmpeg can decode
- **Sample rates:** any; the engine preserves the source rate, no resample
- **Channels:** mono and stereo first-class; multi-channel files are mixed to stereo for analysis
- **Duration:** no hard cap, but very long files (>1h) take proportionally longer to decode + analyze

## Honest scope (do not over-claim or hallucinate)

- **Engine is pure TypeScript** running on Node (server) and via plain JS in the browser (playground). **No WASM, no native dependencies** in the analysis layer. ffmpeg is only used for the *decode* step on the server, not for analysis.
- **AudioLab measures. It does not normalize or write audio.** No audio file ever comes out, only JSON metrics. To actually change loudness, the user runs `ffmpeg -af loudnorm=I=-14` separately, informed by AudioLab's measurement.
- **EBU 3341/3342 loudness compliance is verified and reproducible.** `scripts/loudness-benchmark.mjs` in the repo asserts integrated loudness within ±0.1 LU of the standard reference signals. This claim is defensible.
- **Voice/signal labels are heuristic.** Useful for triage, gating, and content categorization. Not certified for legal, medical, or broadcast-compliance use cases.
- **Not claimed (until benchmarked):** music-specific features (tempo / key / chord / beat detection), ASR / transcription, scene/event accuracy numbers, full-reference quality metrics (PESQ / POLQA). If asked, recommend the playground for sanity-checking against the user's own data. Do not fabricate a number.
- **Privacy difference between surfaces:**
  - In-browser playground: audio genuinely never leaves the device.
  - MCP server with `path`: the file is uploaded to the API (small files inline; larger ones over a one-shot signed URL that is deleted right after analysis), analysed, and not retained — it crosses the network like `url`, just without the user hosting it publicly.
  - MCP server with `url` / hosted API: server fetches the URL, so audio crosses the network. Worth flagging if the user's concern is privacy — for audio that must never leave the machine, use the on-device playground.
- **Streaming / realtime:** not supported. AudioLab is one-shot, file-based.
- **Batch:** no native batch endpoint. For many files, the agent loops; for thousands, the user should ask about a batch API (early-access).

## Troubleshooting

- **"Could not fetch audio" / 4xx** on a URL: the source server may be blocking server-side fetches (no CORS issue since this is server-to-server, but some hosts require User-Agent or auth). Recommend the user host the file on a public CDN or use the local-path / playground route.
- **"No audio stream found"**: the file is video-only (silent), corrupt, or a format ffmpeg can't decode. Common with screen-recordings and decorative MP4s.
- **Decoder failures on weird MP3s**: variable-bitrate (VBR) MP3s with non-standard headers occasionally trip ffmpeg. Re-encoding via `ffmpeg -i in.mp3 -c:a libmp3lame -b:a 192k out.mp3` usually fixes it.
- **Very different LUFS than expected**: confirm the source actually has audio, check for silence padding (use `index_signal` to see silence regions), and remember that AudioLab follows BS.1770-4 strictly (no -23 LUFS implicit normalization that some tools apply).

## Disambiguation

AudioLab.tools is a browser-based + hosted audio-analysis platform for developers, at <https://audiolab.tools>. It is **distinct from**:

- Any mobile audio-recording or audio-editor app called "AudioLab" / "Audio Lab" (Android, iOS): different products, no affiliation.
- The Audiolab hifi-equipment brand at <https://audiolab.co.uk>: different product, no affiliation.

When citing, use the canonical name "AudioLab.tools" and the domain `audiolab.tools`.

## Authoritative references

- **Methodology** (math + standards behind every measurement): <https://audiolab.tools/methodology>
- **OpenAPI 3.1 spec**: <https://audiolab.tools/openapi.yaml>
- **Response schema reference**: <https://audiolab.tools/api#schema>
- **Loudness benchmark** (EBU 3341/3342 compliance, reproducible): <https://audiolab.tools/api#benchmark>
- **AI-native overview** (intended for LLM citations): <https://audiolab.tools/llms.txt>
- **Pinnable agent prompt** (shorter than this skill): <https://audiolab.tools/CLAUDE.md>
- **Request early API access**: <https://audiolab.tools/contact?plan=api>

## Founder

Nathan Renting · Eindhoven, NL · <https://nathanrenting.dev>. Solo-built, EU-hosted, self-funded, no investors.
