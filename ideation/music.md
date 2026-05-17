# Ideation: Music Hearing and Opinions

**Goal:** Enable Nix to fetch music (e.g., from YouTube, local files, streaming), store metadata, and form opinions (like/dislike, genre, mood, etc.) and remember past listening experiences.

## Why
Music is a rich domain for interaction: sharing tracks, discussing moods, generating playlists, and building a shared taste profile between user and agent.

## Possible Components
1. **Music Acquisition**
   - Use `youtube-dl`/`yt-dlp` to download audio from YouTube or other platforms.
   - Support local file playback (mp3, flac, etc.) via `ffmpeg` or direct reading.
   - Optionally integrate with APIs (Spotify, SoundCloud) if credentials available.

2. **Audio Processing & Feature Extraction**
   - Use `ffmpeg` to extract metadata (duration, bitrate, sample rate).
   - Use audio analysis tools (e.g., `librosa` via Python, `essentia`, `audiowaveform`) to extract tempo, key, loudness, spectral features.
   - Generate waveforms or spectrograms for visual sharing (optional).

3. **Memory & Opinion Storage**
   - Maintain a listening history log (JSON or SQLite) with fields: timestamp, source, title, artist, album, duration, user rating, agent mood/tags, notes.
   - Allow agent to store subjective opinions: "I found this track energetic and uplifting", "This song feels melancholic".
   - Implement simple similarity or recommendation based on stored features.

4. **Interaction Interface**
   - Commands: `nix listen <url>` to fetch and play/store a track.
   - `nix opinion <query>` to recall past opinions or ask for opinion on a given track/artist.
   - `nix history` to show recent listening.
   - Integration with wiki: auto-log listening sessions to a music journal.

5. **Safety & Permissions**
   - Respect copyright: only download for personal use, or use royalty-free/creative-commons sources.
   - Allow user to opt-out of storing certain data.

## Next Steps
- Prototype a simple script that takes a YouTube URL, downloads audio, extracts basic metadata via ffmpeg, and stores a JSON entry.
- Add a command to retrieve and summarize stored opinions.
- Experiment with mood tagging via lyrics analysis (if available) or audio features.

## Related
- Wiki: `/home/mataanek/.hermes/wiki/ideation/` for storing ideas.
- Potential skill: `music-opinion` or `music-hearing`.


## Additional Tips (from memory context)
- Wiki Index hub: `wiki/whisky_wiki/index.md`
- Whisky entity pages are organized in the `entities/` directory
- MEMORY: `wux/MEMORY.md`
- Relationship: Wiki Index --[usage]--> MEMORY
