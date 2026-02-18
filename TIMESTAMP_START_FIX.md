# Timestamp Start Time Fix - Extend vs Create New Segment

## Problem

Timestamps were starting at wrong time and duplicating:
```
Audio: 176000 samples (11.0 seconds)
[00:00:03.550 --> 00:00:05.549] And so, my fellow    ← Should start at 0.000!
[00:00:05.549 --> 00:00:07.549] Americans, ask
[00:00:07.549 --> 00:00:09.550] Americans, ask       ← Duplicate!
```

Expected:
```
[00:00:00.000 --> ...] And so, my fellow...
```

## Root Cause

### The Encoder-Token Timing Mismatch

In file mode, the flow is:
1. **Feed all audio** (176,000 samples = 11 seconds)
2. **feed_and_drain() loops** with 16K-sample chunks
3. Each chunk: `vox_stream_feed()` → `drain_tokens()`

**Critical insight**: Multiple encoder runs happen BEFORE tokens are ready!

### Detailed Flow (OLD - Broken)

```
Call 1: feed_and_drain chunk 1
  ├─ vox_stream_feed(16000 samples)
  │   └─ stream_run_encoder()
  │       ├─ Processes mel 0→200
  │       ├─ prev_mel_cursor = 0
  │       ├─ segment_start = 0 * 160 = 0
  │       ├─ segment_end = 200 * 160 = 32000
  │       └─ segment_timestamp_output = 0 (NEW SEGMENT)
  └─ drain_tokens()
      └─ No tokens in queue yet → nothing output

Call 2: feed_and_drain chunk 2
  ├─ vox_stream_feed(16000 samples)
  │   └─ stream_run_encoder()
  │       ├─ Processes mel 200→355
  │       ├─ prev_mel_cursor = 200
  │       ├─ segment_start = 200 * 160 = 32000  ← WRONG! Lost the 0!
  │       ├─ segment_end = 355 * 160 = 56800
  │       └─ segment_timestamp_output = 0 (NEW SEGMENT)
  └─ drain_tokens()
      └─ Still no tokens → nothing output

Call 3: feed_and_drain chunk 3
  ├─ vox_stream_feed(16000 samples)
  │   └─ stream_run_encoder()
  │       ├─ Processes mel 355→555
  │       ├─ prev_mel_cursor = 355
  │       ├─ segment_start = 355 * 160 = 56800 = 3.550s  ← WRONG START!
  │       ├─ segment_end = 555 * 160 = 88800
  │       └─ segment_timestamp_output = 0 (NEW SEGMENT)
  └─ drain_tokens()
      ├─ Tokens NOW in queue!
      └─ Output: [00:00:03.550 --> 00:00:05.549]  ← First timestamp at 3.5s!
```

**Problem**: Each encoder run created a NEW segment, overwriting the previous one. The first segment (starting at 0) was lost before tokens were ready!

## Solution

**Don't create a new segment on each encoder run. Extend the current segment until it's output.**

### New Logic

```c
if (!s->segment_has_timestamp || s->segment_timestamp_output) {
    /* Start NEW segment only if:
     * - No segment exists yet (!segment_has_timestamp)
     * - OR previous segment was output (segment_timestamp_output = 1)
     */
    s->segment_start_sample = prev_mel_cursor * VOX_HOP_LENGTH;
    s->segment_has_timestamp = 1;
    s->segment_timestamp_output = 0;
}
/* Always update end time to include this encoder run's audio */
s->segment_end_sample = s->mel_cursor * VOX_HOP_LENGTH;
```

### Detailed Flow (NEW - Fixed)

```
Call 1: feed_and_drain chunk 1
  ├─ vox_stream_feed(16000 samples)
  │   └─ stream_run_encoder()
  │       ├─ Processes mel 0→200
  │       ├─ !segment_has_timestamp = true → CREATE NEW
  │       ├─ segment_start = 0 * 160 = 0
  │       ├─ segment_end = 200 * 160 = 32000
  │       ├─ segment_has_timestamp = 1
  │       └─ segment_timestamp_output = 0
  └─ drain_tokens()
      └─ No tokens → nothing output

Call 2: feed_and_drain chunk 2
  ├─ vox_stream_feed(16000 samples)
  │   └─ stream_run_encoder()
  │       ├─ Processes mel 200→355
  │       ├─ segment_timestamp_output = 0 → EXTEND (don't create new)
  │       ├─ segment_start unchanged = 0  ← KEPT!
  │       └─ segment_end = 355 * 160 = 56800  ← EXTENDED!
  └─ drain_tokens()
      └─ Still no tokens → nothing output

Call 3: feed_and_drain chunk 3
  ├─ vox_stream_feed(16000 samples)
  │   └─ stream_run_encoder()
  │       ├─ Processes mel 355→555
  │       ├─ segment_timestamp_output = 0 → EXTEND (don't create new)
  │       ├─ segment_start unchanged = 0  ← KEPT!
  │       └─ segment_end = 555 * 160 = 88800  ← EXTENDED!
  └─ drain_tokens()
      ├─ Tokens NOW in queue!
      ├─ Output: [00:00:00.000 --> 00:00:05.550]  ← Correct start!
      └─ segment_timestamp_output = 1

Call 4: feed_and_drain chunk 4
  ├─ vox_stream_feed(16000 samples)
  │   └─ stream_run_encoder()
  │       ├─ Processes mel 555→755
  │       ├─ segment_timestamp_output = 1 → CREATE NEW
  │       ├─ segment_start = 555 * 160 = 88800
  │       ├─ segment_end = 755 * 160 = 120800
  │       └─ segment_timestamp_output = 0
  └─ drain_tokens()
      ├─ Tokens in queue
      └─ Output: [00:00:05.550 --> 00:00:07.550]  ← Next segment!
```

## Key Insight

**Segments should align with TOKEN availability, not encoder runs!**

- Encoder runs multiple times to process all audio
- Tokens come later after decoder processes adapter tokens
- A "segment" for timestamp purposes = audio range that produced the tokens being output
- We accumulate the audio range across multiple encoder runs until tokens appear

## Result

Correct output:
```
Audio: 176000 samples (11.0 seconds)
[00:00:00.000 --> 00:00:05.550] And so, my fellow Americans,
[00:00:05.550 --> 00:00:07.550] ask not what your country
[00:00:07.550 --> 00:00:09.550] can do for you.
[00:00:09.550 --> 00:00:11.000] Ask what you can do for your country.
```

Benefits:
- ✅ First timestamp starts at 0.000
- ✅ No duplicate timestamps
- ✅ Timestamps cover full audio duration
- ✅ Segments align with token output, not encoder runs

## Technical Details

### Segment State Machine

```
State: NO_SEGMENT (segment_has_timestamp = 0)
  └─ Encoder runs → CREATE segment → state: SEGMENT_PENDING

State: SEGMENT_PENDING (segment_has_timestamp = 1, segment_timestamp_output = 0)
  ├─ Encoder runs → EXTEND segment (update end only)
  └─ Tokens output → OUTPUT timestamp → state: SEGMENT_OUTPUT

State: SEGMENT_OUTPUT (segment_has_timestamp = 1, segment_timestamp_output = 1)
  └─ Encoder runs → CREATE NEW segment → state: SEGMENT_PENDING
```

### Why This Works

1. **Preserves first timestamp**: First encoder run creates segment at 0, subsequent runs extend it
2. **No duplicates**: New segment only created after previous one outputs
3. **Accurate boundaries**: Segment accumulates all audio processed until tokens appear
4. **Natural chunking**: Segments naturally align with when decoder produces tokens

## Files Modified

- `voxtral.c`: Modified `stream_run_encoder()` to extend segments instead of always creating new ones
