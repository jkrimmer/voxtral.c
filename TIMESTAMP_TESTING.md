# Timestamp Feature Testing Guide

## Implementation Summary

The timestamp feature adds optional timing information for each transcribed segment. When enabled with `--timestamps`, output shows:

```
[HH:MM:SS.mmm --> HH:MM:SS.mmm] transcription text
```

### Key Implementation Details

1. **Timestamp Calculation**:
   - Based on adapter token positions, not real-time sample feed
   - Each adapter token represents ~80ms of audio (1280 samples at 16kHz)
   - Segment start: `adapter_pos_offset * 1280` samples
   - Segment end: `gen_pos * 1280` samples
   - Time in seconds: `samples / 16000.0`

2. **Segment Boundaries**:
   - New segment starts when decoder (re)starts
   - Segment ends when decoder restarts due to:
     - EOS token
     - KV cache overflow (>2000 entries)
     - Non-text token streak (>64 steps)
     - No-decode watchdog (>20 seconds without tokens)

3. **Output Format**:
   - Timestamp appears before segment's tokens
   - Newline separates segments
   - Compatible with `--alt` flag for alternatives

## Test Cases

### 1. Basic File Transcription

```bash
./voxtral -d voxtral-model -i samples/test_speech.wav --timestamps
```

**Expected**:
- Timestamps in format `[00:00:00.000 --> 00:00:XX.XXX]`
- Start time should be close to 0.000
- End time should match or be close to audio duration
- Timestamps should be monotonically increasing
- Each segment on its own line

**Verify**:
- Check audio duration: `ffprobe samples/test_speech.wav 2>&1 | grep Duration`
- Compare with final timestamp end time

### 2. Streaming from stdin

```bash
ffmpeg -i samples/test_speech.wav -f s16le -ar 16000 -ac 1 - 2>/dev/null | \
    ./voxtral -d voxtral-model --stdin --timestamps
```

**Expected**:
- Similar output to file mode
- Timestamps aligned with audio progression

### 3. With Processing Interval

```bash
./voxtral -d voxtral-model -i samples/test_speech.wav --timestamps -I 1.0
```

**Expected**:
- More frequent segment boundaries (every ~1 second)
- Multiple timestamp lines for short audio

### 4. With Alternative Tokens

```bash
./voxtral -d voxtral-model -i samples/test_speech.wav --timestamps --alt 0.5
```

**Expected**:
- Timestamps before each segment
- Alternative tokens shown inline as `[best|alt1|alt2]`

### 5. Long Audio (stress test)

```bash
ffmpeg -i samples/I_have_a_dream.ogg -f s16le -ar 16000 -ac 1 - 2>/dev/null | \
    ./voxtral -d voxtral-model --stdin --timestamps
```

**Expected**:
- Multiple segments with increasing timestamps
- Final timestamp should be close to ~3 minutes
- Timestamps remain accurate throughout

### 6. Silent Mode with Timestamps

```bash
./voxtral -d voxtral-model -i samples/test_speech.wav --timestamps --silent
```

**Expected**:
- No stderr output
- Only stdout with timestamps and transcription

## Validation Checklist

- [ ] Code builds without errors or warnings
- [ ] Test case 1 passes: basic file with timestamps
- [ ] Test case 2 passes: stdin input
- [ ] Test case 3 passes: custom interval
- [ ] Test case 4 passes: with alternatives
- [ ] Test case 5 passes: long audio
- [ ] Test case 6 passes: silent mode
- [ ] Timestamps are monotonically increasing
- [ ] Timestamp accuracy within 10% of actual audio duration
- [ ] No memory leaks (valgrind on debug build)
- [ ] Works with all backends (BLAS, MPS)

## Manual Verification Steps

1. **Timestamp Accuracy**:
   ```bash
   # Get actual audio duration
   ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 samples/test_speech.wav
   
   # Compare with final timestamp in output
   ./voxtral -d voxtral-model -i samples/test_speech.wav --timestamps | tail -1
   ```

2. **Segment Alignment**:
   - Listen to audio at specific timestamps
   - Verify transcription matches what's spoken at that time

3. **Edge Cases**:
   - Very short audio (<1 second)
   - Audio with long pauses
   - Audio with rapid speech

## Known Limitations

1. Timestamps represent approximate audio position based on adapter tokens
2. Model has inherent delay (default 480ms) before output
3. First segment may have slightly different timing due to initial padding
4. Timestamps granularity is ~80ms (one adapter token)

## Debug Commands

If timestamps seem incorrect:

```bash
# Enable debug output to see encoder/decoder details
./voxtral -d voxtral-model -i audio.wav --timestamps --debug 2>&1 | tee debug.log

# Check mel frame count vs expected
# Expected mel frames = (samples / 160) ≈ (duration_sec * 100)

# Check adapter token count vs expected  
# Expected adapter tokens = mel_frames / 4 (due to downsampling)
```
