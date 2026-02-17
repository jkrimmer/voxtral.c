# Chunk-Based Timestamp Implementation

## User Requirement

Timestamps should reflect the processing chunks, not the entire file duration.

For an 11-second audio file processed with default 2-second interval:
```
[00:00:00.000 --> 00:00:02.000] first chunk transcription
[00:00:02.000 --> 00:00:04.000] second chunk transcription
[00:00:04.000 --> 00:00:06.000] third chunk transcription
...
```

Instead of:
```
[00:00:00.000 --> 00:00:11.000] entire file transcription
```

## Implementation

### Key Concept: Encoder Chunks

The voxtral encoder processes audio in chunks based on the processing interval:
- Default interval: 2 seconds (200 mel frames)
- Each encoder run processes accumulated mel frames
- `mel_cursor` tracks the position in mel frames

### Timestamp Tracking

**In `stream_run_encoder()` function:**

Before encoder processes new mel:
```c
int prev_mel_cursor = s->last_encoder_mel_cursor;
```

After encoder completes:
```c
/* Update timestamp tracking for this encoder chunk.
 * The chunk processed audio from prev_mel_cursor to s->mel_cursor.
 * Convert mel frames to samples: mel_frame * VOX_HOP_LENGTH (160). */
s->segment_start_sample = (int64_t)prev_mel_cursor * VOX_HOP_LENGTH;
s->segment_end_sample = (int64_t)s->mel_cursor * VOX_HOP_LENGTH;
s->last_encoder_mel_cursor = s->mel_cursor;

/* Mark that a new segment timestamp is available.
 * Reset output flag so this chunk's timestamp will be shown. */
s->segment_has_timestamp = 1;
s->segment_timestamp_output = 0;
```

### Calculation Details

**Mel frames to audio samples:**
- Each mel frame = VOX_HOP_LENGTH (160) samples
- Sample rate = 16,000 Hz
- Time in seconds = `(mel_frames * 160) / 16000`

**Example for 2-second chunk:**
- 2 seconds = 32,000 samples
- 32,000 / 160 = 200 mel frames
- Chunk 1: mel 0→200 = samples 0→32,000 = 0.0s→2.0s
- Chunk 2: mel 200→400 = samples 32,000→64,000 = 2.0s→4.0s

## How It Works

### File Mode (All Audio Fed Upfront)

```
1. Feed all audio (11 seconds, 176,000 samples):
   vox_stream_feed(s, samples, 176000)

2. Encoder run 1 (processes mel 0→312):
   → segment_start_sample = 0
   → segment_end_sample = 312 * 160 = 49,920 (~3.1s)
   → First chunk timestamp available

3. Tokens generated and drained:
   → Timestamp [0.0 → 3.1] output with first tokens

4. Encoder run 2 (processes mel 312→512):
   → segment_start_sample = 49,920
   → segment_end_sample = 512 * 160 = 81,920 (~5.1s)
   → New chunk timestamp available

5. More tokens generated and drained:
   → Timestamp [3.1 → 5.1] output with these tokens

... and so on for remaining chunks
```

### Streaming Mode (Incremental Audio)

```
1. Feed first chunk (2 seconds):
   vox_stream_feed(s, samples, 32000)

2. Encoder processes mel 0→200:
   → Timestamp [0.0 → 2.0]
   → Tokens generated for this chunk

3. Feed next chunk (2 more seconds):
   vox_stream_feed(s, samples, 32000)

4. Encoder processes mel 200→400:
   → Timestamp [2.0 → 4.0]
   → Tokens generated for this chunk

... continues as audio arrives
```

## Key Design Decisions

### 1. Timestamps Set by Encoder, Not Decoder

**Rationale**: The encoder determines chunk boundaries based on processing interval. The decoder may restart mid-chunk (EOS, KV overflow), but the timestamp should still reflect the audio chunk that was encoded.

**Implementation**: 
- `stream_run_encoder()` sets timestamps
- `stream_reset_decoder_state()` does NOT clear timestamps
- Timestamps persist across decoder restarts

### 2. Reset Output Flag Per Chunk

Each encoder run sets `segment_timestamp_output = 0` to ensure the new chunk's timestamp is displayed.

### 3. Multiple Timestamps Per File

For a single audio file, multiple timestamp lines appear as encoder processes different chunks. This matches the user's expectation that chunks should be visible in timestamps.

## Processing Interval Impact

The `-I` flag controls chunk size:

```bash
# Default 2-second chunks
./voxtral -d model -i audio.wav --timestamps

# 1-second chunks (more granular)
./voxtral -d model -i audio.wav --timestamps -I 1.0

# 5-second chunks (less granular)
./voxtral -d model -i audio.wav --timestamps -I 5.0
```

**Note**: First chunk is always larger (~312 mel frames ≈ 3.1s) due to model requirements for initial prompt.

## Example Output

For `samples/jfk.wav` (11 seconds) with default settings:

```
./voxtral -d voxtral-model/ -i samples/jfk.wav --timestamps
Loading weights...
Model loaded.
Audio: 176000 samples (11.0 seconds)
[00:00:00.000 --> 00:00:03.120] And so, my fellow Americans,
[00:00:03.120 --> 00:00:05.120] ask not what your country
[00:00:05.120 --> 00:00:07.120] can do for you.
[00:00:07.120 --> 00:00:09.120] Ask what you can do
[00:00:09.120 --> 00:00:11.000] for your country.
```

(Exact boundaries depend on encoder processing and model behavior)

## Files Modified

- `voxtral.c`:
  - Added `last_encoder_mel_cursor` field to `struct vox_stream`
  - Modified `stream_run_encoder()` to track and update chunk boundaries
  - Removed timestamp setting from decoder start
  - Updated `stream_reset_decoder_state()` to preserve timestamps

## Benefits

- ✅ Timestamps reflect actual processing chunks
- ✅ Multiple timestamps for long files
- ✅ Chunk boundaries align with encoder processing
- ✅ Works with any processing interval
- ✅ Consistent for file and streaming modes
- ✅ More informative for users to see how audio was chunked
