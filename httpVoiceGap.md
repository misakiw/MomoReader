# Why the ~3 Second Gap Between Paragraphs with TalkifyTTS

## Overview

When using the online TTS engine "TalkifyTTS" (which proxies Microsoft Edge's free neural voices), there is a noticeable ~3 second gap between paragraphs. There is no single "3-second delay" constant in the code — the gap is the cumulative result of two issues in the playback pipeline.

## Root Causes

### 1. ExoPlayer Buffer Threshold Too High (Primary Cause)

The ExoPlayer was configured with:

```kotlin
DefaultLoadControl.Builder().setBufferDurationsMs(
    1800_000_000,  // minBufferMs — 30 minutes
    1800_000_000,  // maxBufferMs — 30 minutes
    DefaultLoadControl.DEFAULT_BUFFER_FOR_PLAYBACK_MS,                  // 2500ms
    DefaultLoadControl.DEFAULT_BUFFER_FOR_PLAYBACK_AFTER_REBUFFER_MS    // 5000ms
)
```

`DEFAULT_BUFFER_FOR_PLAYBACK_MS` is **2500ms** — ExoPlayer waits until 2.5 seconds of audio is buffered before it starts playing each new media item. `DEFAULT_BUFFER_FOR_PLAYBACK_AFTER_REBUFFER_MS` is **5000ms**. Even when the MP3 file is already on local disk, ExoPlayer still enforces this buffer threshold on every media item transition, adding a mandatory ~2.5s gap between paragraphs.

The `minBufferMs` / `maxBufferMs` of 1.8 billion ms (30 minutes) were also unnecessarily large for a playlist of short TTS audio clips.

### 2. Sequential Download per Paragraph (Secondary Cause)

In the original `downloadAndPlayAudios()`, paragraphs were downloaded one at a time in a single coroutine loop. Each paragraph required a full HTTP round-trip (1–3 seconds). If ExoPlayer finished playing a paragraph before the next one was downloaded and added to the playlist, it would hit `STATE_ENDED`, stop, clear items, and require a full re-prepare cycle.

## Fixes Applied

### Fix 1: Reduce ExoPlayer Buffer Thresholds

Changed the `DefaultLoadControl` to start playback almost immediately:

```kotlin
DefaultLoadControl.Builder().setBufferDurationsMs(
    DefaultLoadControl.DEFAULT_MIN_BUFFER_MS,  // 50s (standard)
    DefaultLoadControl.DEFAULT_MAX_BUFFER_MS,  // 50s (standard)
    100,  // bufferForPlaybackMs: start playback almost immediately
    100   // bufferForPlaybackAfterRebufferMs: same after rebuffer
)
```

Setting `bufferForPlaybackMs` to 100ms means ExoPlayer starts playing as soon as it reads the first chunk from the local MP3 file, instead of waiting 2.5 seconds.

### Fix 2: Parallel Pre-fetching Downloads

Rewrote `downloadAndPlayAudios()` to use a producer-consumer pipeline with up to 3 concurrent downloads:

```
Producer coroutine (IO dispatcher)
    │  For each paragraph:
    │    1. Acquire semaphore permit (max 3 concurrent)
    │    2. Launch download coroutine → CompletableDeferred<fileName>
    │    3. Send deferred into ordered Channel
    │
    ▼
Channel<CompletableDeferred<String>>  (capacity = 3)
    │
    ▼
Consumer (main download coroutine)
    │  For each deferred from channel (in order):
    │    1. Await the deferred
    │    2. Add downloaded file to ExoPlayer as MediaItem
```

While paragraph N is playing, paragraphs N+1 through N+3 download in parallel. By the time ExoPlayer needs the next file, it is already on disk.

### Fix 3: Thread-safe Error Counter

Wrapped all `downloadErrorNo` accesses in `synchronized(downloadErrorLock)` to prevent race conditions from concurrent download coroutines.

## Expected Improvement

| Scenario | Before | After |
|----------|--------|-------|
| ExoPlayer buffer wait per item | 2500ms (default) | 100ms |
| ExoPlayer rebuffer wait | 5000ms (default) | 100ms |
| Download pipeline | Sequential (1 at a time) | Parallel (3 concurrent) |
| Perceived gap between paragraphs | ~3 seconds | Near-zero |

## Files Modified

| File | Changes |
|------|---------|
| `app/src/main/java/io/legado/app/service/HttpReadAloudService.kt` | Reduced ExoPlayer buffer thresholds; rewrote `downloadAndPlayAudios()` with parallel pipeline; added `downloadErrorLock` for thread safety |

## Relevant Files (Unchanged)

| File | Role |
|------|------|
| `app/src/main/java/io/legado/app/service/BaseReadAloudService.kt` | Shared base: audio focus, navigation, content list preparation |
| `app/src/main/java/io/legado/app/model/analyzeRule/AnalyzeUrl.kt` | HTTP request construction, rate limiting |
| `app/src/main/java/io/legado/app/help/ConcurrentRateLimiter.kt` | Concurrent rate limiting between requests |
| `app/src/main/java/io/legado/app/data/entities/HttpTTS.kt` | Data model for online TTS engine configuration |
