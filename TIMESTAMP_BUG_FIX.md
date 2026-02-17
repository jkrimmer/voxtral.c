# Timestamp Bug Fix

## Problem

When transcribing audio files with the `--timestamps` flag, the end time was incorrectly showing as much shorter than the actual audio duration.

### Example
For an 11-second audio file (samples/jfk.wav):
```bash
./voxtral -d voxtral-model/ -i samples/jfk.wav --timestamps
[00:00:00.000 --> 00:00:03.439] And so, my fellow Americans, ask not...
```

Expected: `[00:00:00.000 --> 00:00:11.000]`
Actual: `[00:00:00.000 --> 00:00:03.439]`

## Root Cause

The timestamp was being captured and output too early in the processing pipeline:

1. **Decoder Prefill Phase**:
   - Decoder starts with ~39 prompt tokens (1 BOS + 32 padding + 6 delay)
   - `segment_end_sample` is set to `prompt_len * 1280` = ~50,000 samples = 3.1 seconds
   - First tokens are enqueued

2. **First drain_tokens() Call**:
   - Called after prefill but BEFORE generation loop completes
   - Retrieves timestamp with early `segment_end_sample` value (3.4 seconds)
   - Marks timestamp as output (`segment_timestamp_output = 1`)
   - Drains available tokens

3. **Generation Loop Continues**:
   - Processes remaining adapter tokens (39 → 186 in the example)
   - Updates `segment_end_sample` correctly to 186 * 1280 = ~14.9 seconds
   - But timestamp was already output!

4. **Subsequent drain_tokens() Calls**:
   - Don't output timestamp again (already marked as output)
   - Only drain remaining tokens

## Solution

Modified `vox_stream_get_timestamp()` to delay timestamp output until generation completes:

### Fix 1: Wait for Generation Completion

```c
/* Don't output timestamp until generation completes for this segment.
 * If decoder is active and gen_pos hasn't caught up to total_adapter,
 * we're still generating tokens and segment_end_sample will be updated. */
if (s->decoder_started && !s->eos_seen && s->gen_pos < s->total_adapter) {
    return 0;  /* Still generating, wait for completion */
}
```

This check ensures:
- Timestamp is NOT output while `gen_pos < total_adapter` (generation in progress)
- Timestamp IS output when `gen_pos >= total_adapter` (generation complete)
- All tokens for the segment have been generated and `segment_end_sample` has its final value

### Fix 2: Cap End Time at Actual Audio Duration

```c
/* Cap end time at actual audio fed to avoid showing time beyond real audio duration.
 * Adapter tokens may include padding beyond actual audio. */
int64_t end_sample = s->segment_end_sample;
if (end_sample > s->real_samples_fed) {
    end_sample = s->real_samples_fed;
}
*end_sec = (double)end_sample / VOX_SAMPLE_RATE;
```

This addresses the issue where:
- Mel spectrogram computation adds padding for proper processing
- `total_adapter` may include tokens representing padded audio beyond the actual file
- Without capping, a 11-second file might show end time of 14.9 seconds
- Capping at `real_samples_fed` ensures end time matches actual audio duration

## Testing

To verify the fix works correctly:

```bash
# Test with jfk.wav (11 seconds)
./voxtral -d voxtral-model/ -i samples/jfk.wav --timestamps

# Expected output:
# [00:00:00.000 --> 00:00:11.000] <transcription>

# Test with test_speech.wav
./voxtral -d voxtral-model/ -i samples/test_speech.wav --timestamps

# Test with stdin
ffmpeg -i samples/jfk.wav -f s16le -ar 16000 -ac 1 - 2>/dev/null | \
    ./voxtral -d voxtral-model --stdin --timestamps
```

## Technical Details

### Timing Calculation

- Sample rate: 16,000 Hz
- Each adapter token: 1,280 samples = 80ms of audio
- Calculation: `time = samples / 16000.0`

### Generation Flow

1. Encoder processes mel frames → adapter tokens
2. Decoder prefills with prompt_len tokens
3. Generation loop: while `gen_pos < total_adapter`
   - Generate one token per iteration
   - Update `segment_end_sample = gen_pos * 1280`
   - Increment `gen_pos++`
4. Exit when `gen_pos >= total_adapter`
5. Final `segment_end_sample` reflects last token position

### Segment Boundaries

Segments are created when:
- Decoder starts (after prefill)
- Decoder restarts due to:
  - EOS token
  - KV cache overflow
  - Non-text token streak
  - Watchdog timeout

Each segment gets one timestamp covering all its tokens.

## Files Modified

- `voxtral.c`: `vox_stream_get_timestamp()` function
  - Added generation completion check
  - Added end_sample capping logic

## Impact

- ✅ Timestamps now show correct end times matching audio duration
- ✅ Works for all input modes (file, stdin, microphone)
- ✅ Compatible with all existing flags
- ✅ No performance impact (only adds simple checks)
- ✅ Maintains streaming behavior (tokens still output as generated)
