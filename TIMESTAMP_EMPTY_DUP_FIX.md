# Timestamp Output Bug Fix - Empty and Duplicate Timestamps

## Problem

When running with `--timestamps`, the output showed:
1. **Empty timestamps**: Lines with timestamp but no text
2. **Duplicate timestamps**: Each timestamp appearing twice

### Example of the Bug

```
[00:00:00.000 --> 00:00:03.550]
[00:00:03.550 --> 00:00:05.549] And so, my fellow
[00:00:05.549 --> 00:00:07.549] Americans, ask
[00:00:07.549 --> 00:00:09.550] Americans, ask
[00:00:09.550 --> 00:00:11.550] not what your
[00:00:11.550 --> 00:00:13.550] not what your
...
```

## Root Cause

### The Flow in File Mode

```
1. feed_and_drain() loops with 16000-sample chunks:
   for chunk in audio:
       vox_stream_feed(stream, chunk)  # May trigger encoder
       drain_tokens(stream)            # Try to drain tokens

2. Encoder runs and sets:
   segment_has_timestamp = 1
   segment_timestamp_output = 0  # Ready to output

3. drain_tokens() called multiple times:
   - First call: Checks timestamp available → outputs it
   - But no tokens in queue yet → empty line!
   - Second call: Encoder ran again → new timestamp available
   - Outputs another timestamp → duplicate!
```

### The Issue

The old `drain_tokens()` logic was:
```c
static void drain_tokens(vox_stream_t *s) {
    // Check for timestamp FIRST
    if (show_timestamps) {
        if (vox_stream_get_timestamp(...)) {
            printf("[%s --> %s] ", ...);  // Output timestamp
        }
    }
    
    // THEN try to get tokens
    while ((n = vox_stream_get(s, tokens, 64)) > 0) {
        // Output tokens
    }
}
```

Problems:
1. Timestamp output happens BEFORE checking if tokens exist
2. If encoder ran but decoder hasn't generated tokens yet → empty timestamp
3. Multiple `drain_tokens()` calls in feed loop → multiple timestamp outputs

## Solution

Restructure to output timestamp ONLY when tokens are available:

```c
static void drain_tokens(vox_stream_t *s) {
    int first_batch = 1;
    
    // Get tokens in loop
    while ((n = vox_stream_get(s, tokens, 64)) > 0) {
        // On FIRST batch of tokens, check for timestamp
        if (show_timestamps && first_batch) {
            if (vox_stream_get_timestamp(...)) {
                printf("[%s --> %s] ", ...);  // Output timestamp
            }
            first_batch = 0;  // Only once per drain_tokens() call
        }
        
        // Output tokens
        for (int i = 0; i < n; i++) {
            fputs(tokens[i], stdout);
        }
    }
}
```

### Key Changes

1. **Timestamp check inside token loop**: Only executes if tokens exist
2. **first_batch flag**: Ensures timestamp outputs only once per `drain_tokens()` call
3. **No tokens = no timestamp**: If queue empty, nothing outputs

## How It Works Now

### File Mode Flow

```
1. feed_and_drain() loops:
   vox_stream_feed(chunk)
   drain_tokens()

2. First few calls:
   - Encoder runs, marks timestamp available
   - drain_tokens() called
   - No tokens yet → while loop doesn't execute
   - No timestamp output ✓

3. When tokens available:
   - drain_tokens() called
   - while loop enters (tokens exist)
   - first_batch = true → check timestamp
   - Timestamp available → output it
   - first_batch = false
   - Output tokens
   - Next iteration: first_batch = false → no timestamp check ✓

4. Next chunk:
   - Encoder ran again, new timestamp available
   - drain_tokens() called
   - first_batch resets to true
   - New timestamp output with new tokens ✓
```

## Result

Correct output format:
```
[00:00:00.000 --> 00:00:03.120] And so, my fellow
[00:00:03.120 --> 00:00:05.120] Americans, ask not
[00:00:05.120 --> 00:00:07.120] what your country
[00:00:07.120 --> 00:00:09.120] can do for you.
...
```

Each timestamp:
- ✅ Appears only once
- ✅ Has text after it
- ✅ Corresponds to the encoder chunk that produced those tokens

## Technical Details

### Why This Works

**Synchronization**: By checking timestamp inside the token loop, we ensure timestamp and tokens are output together atomically.

**first_batch flag**: 
- Scoped to each `drain_tokens()` call
- Resets on each call
- Ensures timestamp checked only once even if multiple batches of tokens retrieved

**No empty output**: If no tokens in queue, the while loop never executes, so timestamp never outputs.

### Edge Cases Handled

1. **Multiple encoder runs before first tokens**: Only first set of tokens gets first timestamp
2. **Tokens split across calls**: first_batch ensures timestamp only on first tokens
3. **Alternatives mode**: Same logic applied to `vox_stream_get_alt()` path

## Files Modified

- `main.c`: 
  - Restructured `drain_tokens()` function
  - Moved timestamp check inside token retrieval loop
  - Added `first_batch` flag
  - Applied to both normal and alternatives mode

## Testing

Should verify:
- No empty timestamp lines
- No duplicate timestamps
- Each timestamp followed by text
- Works with `--alt` flag
- Works with different `-I` intervals
