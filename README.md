# WebKit Blob Stream Bug Reproduction

Minimal test case demonstrating a crash in iOS WebKit when using ReadableStream with Blob slices.

## The Bug

iOS WebKit crashes when reading large Blobs using this pattern:

```javascript
// Create slice from offset to END of file
const slice = blob.slice(startPos);
const reader = slice.stream().getReader();

// Read repeatedly until target position
while (currentPos < targetPos) {
    const { done, value } = await reader.read();
    currentPos += value.length;
}
```

When multiple workers (up to 4) run this pattern **in parallel**, it creates massive memory pressure:
- For a 200MB file with 4 parallel workers
- Each worker creates a slice to file end: ~200MB, ~192MB, ~184MB, ~176MB
- Total: ~750MB of slice references for a 200MB file
- **Result: Page crashes on iOS**

## Workaround

Using `arrayBuffer()` with exact ranges works fine:

```javascript
const buffer = await blob.slice(startPos, endPos).arrayBuffer();
```

## How to Test

```bash
npm run dev
```

1. Access the network URL on an iOS device
2. Upload a large video file (200MB+)
3. Try **Stream API** mode - crashes on iOS
4. Try **ArrayBuffer API** mode - works fine

## Platforms Affected

- **Crashes:** iOS Safari, iOS Chrome, iOS Firefox (all iOS browsers use WebKit)
- **Works:** macOS Safari, Chrome desktop, Firefox desktop

## Related

- [Mediabunny Issue #184](https://github.com/Vanilagy/mediabunny/issues/184)
- Source: [BlobSource implementation](https://github.com/Vanilagy/mediabunny/blob/main/src/source.ts#L222)
