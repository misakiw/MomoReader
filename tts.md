# TTS (Text-to-Speech) Architecture

Legado supports two TTS modes: **local TTS** using Android's built-in engine, and **HTTP TTS** using online speech synthesis APIs. Both share a common base service for playback control, audio focus, and media session integration.

## Architecture Overview

```
ReadAloud (coordinator)
    └── BaseReadAloudService (abstract)
            ├── TTSReadAloudService  — local TTS (Android system)
            └── HttpReadAloudService — online TTS (HTTP APIs)
```

- [ReadAloud](/app/src/main/java/io/legado/app/model/ReadAloud.kt) — entry point that selects and launches the appropriate service
- [BaseReadAloudService](/app/src/main/java/io/legado/app/service/BaseReadAloudService.kt) — shared logic (audio focus, media session, notifications, navigation)
- [TTSReadAloudService](/app/src/main/java/io/legado/app/service/TTSReadAloudService.kt) — local TTS via `android.speech.tts.TextToSpeech`
- [HttpReadAloudService](/app/src/main/java/io/legado/app/service/HttpReadAloudService.kt) — online TTS via HTTP + ExoPlayer
- [HttpTTS](/app/src/main/java/io/legado/app/data/entities/HttpTTS.kt) — data model for online TTS engine configuration
- [TTS](/app/src/main/java/io/legado/app/help/TTS.kt) — lightweight standalone helper for short text (e.g. dictionary lookups)

## Engine Selection

`ReadAloud.getReadAloudClass()` decides which service to start based on the `ttsEngine` setting:

| Setting value | Service used |
|---|---|
| Blank / null | `TTSReadAloudService` (local) |
| Numeric ID matching an `HttpTTS` record | `HttpReadAloudService` (online) |
| Numeric ID with no matching record | `TTSReadAloudService` (fallback) |

Each book can override the global TTS engine via `book.getTtsEngine()`.

## Local TTS — `TTSReadAloudService`

Uses Android's `android.speech.tts.TextToSpeech` API.

### Initialization

- Creates a `TextToSpeech` instance, optionally passing a specific engine package name (e.g. Google TTS, Samsung TTS).
- Implements `TextToSpeech.OnInitListener`; playback starts only after `onInit` reports `SUCCESS`.

### Speaking

- Iterates through `contentList` (paragraphs of the current chapter).
- First paragraph: `QUEUE_FLUSH` (replaces anything currently queued).
- Subsequent paragraphs: `QUEUE_ADD` (appended to the queue).
- Paragraphs matching `notReadAloudRegex` (punctuation-only) are skipped.

### Progress Tracking

An `UtteranceProgressListener` receives callbacks:

- `onStart` — updates page index and TTS progress indicator.
- `onRangeStart` — fine-grained position tracking within an utterance; advances the page when the spoken position crosses a page boundary.
- `onDone` — moves to the next paragraph; triggers next chapter when all paragraphs are done.
- `onError` — skips the failed paragraph and continues.

### Speech Rate

- If `ttsFlowSys` is enabled, the system default rate is used (engine is reinitialized on rate change).
- Otherwise, a custom rate is set via `TextToSpeech.setSpeechRate()` using the formula `(AppConfig.ttsSpeechRate + 5) / 10f`.

### Error Recovery

If `tts.speak()` returns `TextToSpeech.ERROR`, the engine is torn down (`stop()` + `shutdown()`) and reinitialized.

## HTTP TTS — `HttpReadAloudService`

Uses configurable HTTP endpoints that return audio streams, played back with ExoPlayer (Media3).

### How It Works

1. For each paragraph, sends the text to the URL defined in `HttpTTS.url` via `AnalyzeUrl`.
2. The server returns an audio stream (typically MP3).
3. ExoPlayer plays the audio files sequentially.

### Two Playback Modes

| Mode | Behavior |
|---|---|
| Download-first (default) | Downloads all audio to disk as `.mp3` files, then adds them as `MediaItem`s to ExoPlayer. |
| Streaming (`streamReadAloudAudio`) | Uses `CacheDataSource` so ExoPlayer streams and caches audio on the fly. |

### Caching

- Audio files are named using MD5 hashes of: chapter title + TTS URL + speech rate + text content.
- Streaming mode uses a 128 MB LRU cache (`SimpleCache`).
- Old cache files are cleaned up after 10 minutes of inactivity.

### Pre-downloading

After the current chapter's audio is queued, the service pre-downloads up to 10 paragraphs of the next chapter to reduce latency.

### Speech Rate

The rate is passed as a `speakSpeed` parameter in the HTTP request. The remote server is responsible for adjusting the audio speed.

### Error Handling

- Socket timeouts and connection errors are retried up to 5 times before pausing.
- Other download errors fall back to a silent audio placeholder.
- ExoPlayer playback errors skip to the next media item; 5 consecutive errors pause reading.

## Common Base — `BaseReadAloudService`

Shared functionality for both TTS modes:

- **Audio focus** — requests focus before playing, pauses on `AUDIOFOCUS_LOSS`, resumes on `AUDIOFOCUS_GAIN`. Can be disabled via `ignoreAudioFocus` setting.
- **Phone call handling** — listens for `CALL_STATE_RINGING` to pause, resumes on `CALL_STATE_IDLE`.
- **MediaSession** — exposes playback state and controls to the system (lock screen, notification, Bluetooth headset buttons).
- **Notification** — foreground service notification with prev/play-pause/stop/next/timer controls.
- **Navigation** — prev/next paragraph (`prevP`/`nextP`), prev/next chapter, with position tracking.
- **Sleep timer** — countdown in minutes, auto-stops reading when it reaches zero.
- **Wake lock** — optional `PARTIAL_WAKE_LOCK` + Wi-Fi lock to prevent the device from sleeping during playback.

## Standalone TTS Helper — `TTS.kt`

A simpler wrapper around `TextToSpeech` for use outside the read-aloud service (e.g. dictionary word pronunciation):

- Auto-initializes on first `speak()` call.
- Splits text by newlines and queues each segment.
- Auto-releases the TTS engine after 60 seconds of inactivity.
- Exposes a `SpeakStateListener` interface for start/done callbacks.
