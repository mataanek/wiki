# Ideation: Home Speaker Integration (Apple HomePod/HomeKit)

**Goal:** Enable Nix to send and receive audio through the user's Apple Home smart speakers (HomePod, HomePod mini, etc.) for seamless voice interaction—allowing the agent to speak responses aloud and listen for wake words/commands via the home speaker array.

## Why
- **Natural interaction:** Talk to Nix from anywhere in the house without needing a phone or computer nearby.
- **Ambient computing:** Turn the home into an interactive environment where the agent can proactively share information, reminders, or engage in dialogue.
- **Accessibility:** Hands-free operation useful while cooking, cleaning, or other activities.
- **Enhanced presence:** Makes the agent feel more "embedded" in the living space rather than confined to a terminal/chat interface.

## Technical Approaches & Challenges
*Note: Apple's ecosystem is highly restricted; direct third-party agent control of HomePods for arbitrary audio I/O is not officially supported. All approaches involve workarounds with varying reliability.*

### 1. **Audio Output (Speaking via HomePods)**
   - **AirPlay Streaming:**
     - Use `airplay` or `raop` tools (e.g., `shairport-sync` as receiver, or `pyatv`/`pyairplay` as sender) to stream audio from Hermes agent to an AirPlay-compatible device.
     - *Challenge:* Requires the HomePod to be discoverable as an AirPlay target (usually is by default). Latency may be noticeable for real-time conversation.
     - *Feasibility:* Moderate—can play pre-recorded TTS or stream live audio, but buffering may cause delays.

   - **Siri Shortcuts + Webhooks:**
     - Create a HomeKit Shortcut triggered by a webhook (via Home Assistant or similar) that speaks text using Siri.
     - *Challenge:* Indirect and clunky; relies on Siri's voice (not Nix's voice), and requires exposing a local webhook securely.

   - **HomeBridge Plugin:**
     - Develop or use a HomeBridge plugin that exposes a "speaker" service accepting text-to-speech commands.
     - *Challenge:* HomeBridge runs on macOS/Linux but requires maintaining a separate hub; Apple may break compatibility with updates.

### 2. **Audio Input (Listening via HomePods)**
   - **Major Hurdle:** Apple does *not* allow third-party apps to access HomePod microphones for arbitrary audio processing. Siri is the only authorized voice interface.
   - **Workarounds (all imperfect):**
     - **Siri Proxy:** Intercept Siri requests via a man-in-the-middle proxy (e.g., `Siriproxy` or custom DNS/SSL setup) to capture user speech after "Hey Siri" is invoked.
       - *Risks:* Security concerns (MITM), Apple actively blocks such proxies, violates ToS.
     - **Use HomePod as AirPlay Receiver for Mic:** Theoretically, route mic input *to* the agent via AirPlay (reverse of output), but HomePods do not expose mic streams this way.
     - **Fallback to Phone/Watch:** Have the user invoke Siri on their iPhone/Apple Watch ("Hey Siri, ask Nix...") where a Shortcut forwards the text to the agent via webhook.
       - *Pros:* Uses authorized Siri path; *Cons:* Requires device in hand, loses ambient "whole-house" feel.
     - **Dedicated Always-On Mic:** Place a separate, always-listening device (e.g., Raspberry Pi with ReSpeaker) in the room, bridging to Hermes agent—but this defeats the purpose of using *existing* HomePods.

### 3. **Integration Layer**
   - **Middleware Required:** Likely need a lightweight service running on a always-on device (Mac, Linux box, or Pi) that:
     - Exposes a local API for Hermes to send text (for TTS output via AirPlay).
     - Listens for incoming audio triggers (from workaround mic sources) and forwards text to Hermes.
     - Handles state (e.g., "listening mode", volume control).
   - **Technology Options:**
     - Python server using `pyatv` for AirPlay out, `webrtcvad` or `snowboy` for wake-word detection on alternative mic input.
     - Or leverage existing smart home hubs (Home Assistant) via their REST/WebSocket APIs if already deployed.

## Safety, Privacy & Practical Notes
- **Privacy:** Streaming audio to/from HomePods via unofficial means risks exposing sensitive conversations. Any solution must prioritize local processing—no audio should leave the home network unencrypted.
- **Reliability:** Apple frequently updates security; any workaround may break with OS updates. Factor in maintenance overhead.
- **User Experience:** True duplex conversation (natural back-and-forth) is hard due to latency and half-duplex limitations of many audio streaming methods. Push-to-talk or defined interaction phrases may be more realistic.
- **Alternative:** If Apple Home integration proves too fragile, consider using a cross-platform smart speaker (e.g., Sonos with AirPlay 2, or a generic Bluetooth speaker) as a more open substitute—but this doesn't fulfill the *specific* request to use *existing* Apple Home speakers.

## Next Steps (Ideation Only - Not for Immediate Implementation)
1. **Research:** Survey current state of `pyatv`, `shairport-sync` reverse-engineering efforts, and HomeBridge speaker plugins.
2. **Prototype Output:** Test sending TTS audio to a HomePod via `ffmpeg` + `paplay` (if PulseAudio supports AirPlay) or `aptx` tools.
3. **Explore Input Alternatives:** Investigate if any macOS accessibility features allow routing system audio (including mic) to a virtual AirPlay sink.
4. **Define Interaction Model:** Decide whether to pursue:
   - Push-to-talk (user presses button to initiate listening)
   - Wake-word on alternative hardware (with HomePod as output only)
   - Siri Shortcut-triggered text exchange (least ideal but most compliant)
5. **Document Assumptions:** Note that full-duplex, always-listening HomePod interaction via unofficial means is likely unsustainable long-term; consider framing this as a "experimental" or "proof-of-concept" goal.

## Prototype Logs (2026-05-17)
- **TTS Pipeline:** Successfully tested Hermes neuTTS voice control skill. Generated audio file at `/home/mataanek/.hermes/audio_cache/tts_20260517_103959.ogg` and played locally via `paplay`.
- **AirPlay Discovery:** Attempted to discover AirPlay devices from WSL using `pyatv` (version 0.17.0) with a 5-second timeout. Found 0 devices. Likely due to network isolation between WSL2 and the host's Wi-Fi interface (AirPlay relies on multicast DNS which may not be bridged).
- **Network Environment:** WSL2 eth0 IP: 172.22.243.32/20. Host Windows IP (default gateway): 172.22.240.1. Multicast traffic may not be forwarded by the WSL2 virtual switch.

## Assumptions & Sustainability (Updated 2026-05-17)
- Apple’s ecosystem does not permit third‑party mic access; any solution relying on it is experimental and may break with OS updates.
- AirPlay output is publicly supported but may introduce latency (~200‑400 ms) and buffering delays. Requires multicast DNS (mDNS) to be functional between the agent and the HomePod.
- Local processing (wake word, STT, TTS) ensures privacy; no audio leaves the home unencrypted.
- Hardware fallback (e.g., Raspberry Pi + ReSpeaker) incurs cost (~$50‑$80) but provides a stable, compliant platform for wake-word detection and audio I/O.
- Maintenance overhead: monitor GitHub projects for updates to `pyatv`, `shairport-sync`, `openWakeWord`, `whisper`; schedule quarterly review.
- In WSL2, AirPlay discovery may require enabling multicast bridging or using the host's network stack. Alternative: relay audio via shared folder to Windows host and use a Windows-based AirPlay sender (e.g., iTunes, Airfoil, or open-source `shairport-sync` on Windows).

## Related
- Wiki Index: `wiki/whisky_wiki/index.md`
- Existing ideation: `/home/mataanek/.hermes/wiki/ideation/` (music, training-advisory, etc.)
- Memory: `wux/MEMORY.md` (may contain user preferences on voice/interaction style)
- Hermes Agent Capabilities: Current TTS (`text_to_speech`) and audio input (via terminal `rec` or `arecord`) are available but limited to local machine.