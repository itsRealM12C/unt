# Unity Asset Extractor — Technical README

A single-file, dependency-free (except an optional Three.js CDN import for the 3D preview) HTML
tool that unpacks Unity `_Data` folder archives entirely client-side: no upload, no server, no
build step. This document explains *how* it works at the binary level, not just what it does —
each format below was reverse-engineered from raw bytes and cross-validated against ground truth
(UnityPy, `texture2ddecoder`, native `libvorbis`, and the system `unzip` utility) before shipping.

---

## 1. Architecture

Everything lives in one `<script>` block, organized into independent modules that share only
primitive data structures (`Uint8Array`, plain objects):

```
zip.js + inflate.js   → pure-JS DEFLATE decoder + zip central-directory reader/writer
fsb5.js                → FMOD Sample Bank (FSB5) container parser
vorbis_rebuild.js       → Ogg Vorbis stream reconstruction (bitpacker, muxer, CRC)
unity_scan.js           → Unity SerializedFile heuristic scanner (AudioClip)
texture.js              → Texture2D scanner + pixel decoders (uncompressed + DXT)
mesh.js                 → Unity Mesh binary reader + OBJ exporter + Three.js viewer
```

There is **no dependency on a full Unity TypeTree parser**. Building one (à la AssetStudio /
UnityPy) means implementing a generic recursive descent binary reader driven by a serialized
type-tree metadata section — version-dependent, large in scope, and overkill for this tool's
goal. Instead, every object format below was reverse-engineered by:

1. Loading a real `.assets` file with UnityPy (which *does* have a full TypeTree engine) to get
   ground-truth field values.
2. Dumping the object's raw bytes and manually correlating byte offsets with those known values.
3. Writing a **heuristic scanner**: search for length-prefixed name strings, then read fields at
   validated offsets/gaps relative to the name, with sanity bounds on every read (dimensions,
   counts, format enums) so a false match fails fast instead of producing garbage.
4. Re-running the JS implementation against the *entire* object population in a real file and
   diffing every field against UnityPy, until it's 100% byte-exact — not "looks right."

This trades generality (a new Unity major version *could* shift field layouts) for zero
dependencies and full auditability — every offset in this codebase has a comment explaining how
it was derived and what percentage of a real dataset it matched.

---

## 2. Zip layer

Game data ships as a zip (`Game_Data.zip` or similar). Rather than pull in a zip library:

- **Central directory reader** (`parseZip`): walks the End-Of-Central-Directory record backward
  from EOF (searching for the `PK\x05\x06` signature), then the central directory itself, to
  build a `{filename: {compMethod, compSize, uncompSize, localHeaderOffset}}` map without
  decompressing anything yet — cheap even on large archives.
- **DEFLATE decoder** (`inflateRaw`): a from-scratch RFC 1951 implementation (fixed + dynamic
  Huffman blocks, stored blocks, LZ77 back-references) writing into a **growable `Uint8Array`**
  rather than a JS array of numbers — the first version used array-push-per-byte and took 3.4s
  for a 10 MB entry; switching to a pre-sized, doubling-capacity `Uint8Array` buffer dropped that
  to ~100ms (32× faster, and avoids ~8× memory overhead from boxed number arrays).
- **Zip writer** (`buildZip`, used by the "Zip Model / Zip Texture / ... / Zip All" buttons):
  store-only (no re-compression needed — the outputs are already-compressed PNG/OGG/etc.), with
  a proper local-file-header + central-directory + EOCD structure and a **standard IEEE 802.3
  reflected CRC-32** (`0xEDB88320` polynomial) — note this is a *different* CRC-32 variant from
  the one described in §4 for Ogg pages, which is non-reflected. Verified by round-tripping
  through both this tool's own reader and the system `unzip -t`.

---

## 3. Unity `SerializedFile` header

Every `.assets` file starts with a version-dependent header. For the modern "large file" format
(version ≥ 22, used by all Unity 2020+ projects tested against):

```
u32  metadataSize (legacy, 0 if large-file format)
u32  fileSize      (legacy, 0)
u32  version        (big-endian! e.g. 22)
u32  dataOffset     (legacy, 0)
u64  metadataSize   (big-endian)
u64  fileSize
u64  dataOffset
u64  unknown
[metadata: Unity version string, target platform, type tree, object table...]
```

All header integers are **big-endian**, unlike everything that follows in the object data itself
(little-endian). This tripped up the very first parsing attempt — the header fields were being
read at the wrong byte offsets until cross-checking `fileSize`/`metadataSize`/`dataOffset`
against the file's actual length and the known start of readable ASCII (`"2020.3.25f1"`) pinned
down the correct 48-byte header layout.

The heuristic scanners below **don't** parse this header or the type tree — they scan the whole
byte range for recognizable object patterns. This is deliberately robust to metadata-section
version drift, at the cost of needing per-object-type sanity checks to avoid false positives.

---

## 4. Audio: FSB5 + Ogg Vorbis reconstruction

### 4.1 Locating the AudioClip → FSB5 blob

Unity's `AudioClip` object serializes (relevant fields, in order): name string, several
scalar fields (channels, frequency, bits-per-sample, length, flags), then a `StreamingInfo`
struct: **source filename string → u64 offset → u64 size**. The scanner:

1. Finds all length-prefixed ASCII strings in the `.assets` buffer.
2. For each candidate *name* string, looks within a small trailing window for a *second* string
   matching a resource-file extension (`.resource`, `.resS`, ...) — this is the `m_Source` field.
3. Reads the following 16 bytes as `(u64 offset, u64 size)`.
4. **Validates by checking the referenced resource file's bytes at that offset literally start
   with the `FSB5` magic number** — the strongest possible signal, eliminating false positives
   even when byte-searching a multi-megabyte file for a name+offset+size triple.

### 4.2 FSB5 container parsing

FSB5 (FMOD Sample Bank v5) header: magic, version, sample count, header/name-table/data sizes,
codec mode. Each sample header is a **bit-packed 64-bit word** (next-chunk flag, frequency index,
channel count, 28-bit data offset ×16, 30-bit sample count), optionally followed by variable
metadata chunks (channel override, frequency override, and critically, `VORBISDATA` — a CRC32 of
the encoder's setup parameters).

### 4.3 Rebuilding a standard Ogg Vorbis stream

FSB5 stores Vorbis audio **without** the identification/comment/setup headers a standard `.ogg`
file needs — FMOD strips them at build time since they're mostly redundant across many clips
using the same encoder settings, keyed by that `VORBISDATA` CRC32. To play the audio in a browser,
this tool:

1. **Rebuilds the identification header** from scratch (channels, sample rate, block sizes) using
   a hand-written LSB-first bit-packer matching libogg's `oggpack_write` convention.
2. **Rebuilds the comment header** with a placeholder vendor string (any valid header works).
3. **Looks up the setup header** in an embedded table of ~164 precomputed setup packets, keyed by
   the same CRC32 FSB5 stores — this table was generated once, offline, by calling
   `libvorbisenc`'s real `vorbis_encode_setup_init` for every known `(quality, channels, rate)`
   combination FSB5's format uses and capturing the resulting setup packet bytes.
4. **Computes granule positions** per audio packet. This requires knowing each packet's block
   size (short/long), which normally means parsing the setup header's mode list — but standard
   libvorbis encoder output always defines exactly 2 modes (short=0, long=1) selected by a single
   bit, so the block size can be read directly from bit 1 of each packet's first byte without
   implementing a floor/residue/codebook decoder.
5. **Pages everything into Ogg containers** with a hand-written non-reflected CRC-32
   (`0x04C11DB7` polynomial, direct/MSB-first — the "other" CRC-32 variant, distinct from zip's)
   and proper lacing-value segment tables, following the standard 2-page header convention
   (id-header alone on page 0, comment+setup together on page 1, forced flush on both).

**Validation:** the rebuilt Ogg file was byte-for-byte identical to a reference rebuild using the
real `python-fsb5` library (which itself shells out to libvorbis) — 0 sample differences across
a full waveform.

PCM8/16/32 and MP3 (raw MPEG frame passthrough) are handled separately and more simply — just a
WAV header wrap or direct passthrough, no reconstruction needed.

---

## 5. Textures: streamed vs. inline, and DXT block decompression

Unity's `Texture2D` stores pixel data in one of two ways depending on size, and both needed
independent reverse-engineering since the trailing struct layout differs:

### 5.1 Streamed (`m_StreamData`)

Layout: `u64 offset → u32 size → path string` (note: opposite field order from AudioClip's
`string → u64 → u64`, and a 32-bit size instead of 64-bit — these are genuinely different
serialized structs despite superficial similarity). The path string names an external
`.resS`/`.resource` file. Validated: the *path* field is checked against the resource file
actually being scanned (see §7) to prevent cross-pairing when a zip contains many similarly-sized
resource files.

### 5.2 Inline (small textures embedding pixel data directly in `.assets`)

No streaming path at all — instead, at a **fixed 84-byte gap** past the (4-byte-aligned) end of
the name string, there's a `u32 length` immediately followed by raw pixel bytes. This offset was
derived by binary-searching a known pixel-data byte sequence (from UnityPy's decoded
`image_data`) within the raw object bytes across 41 real inline textures, confirming the gap is
constant regardless of image dimensions (since only the *value* varies, not the surrounding
struct's byte layout). An earlier version used a "search forward for a length-prefixed blob
matching the expected computed size" approach instead of the fixed offset — this produced a
**false positive** on one texture where an unrelated earlier field happened to numerically equal
the real data length. The fixed-offset approach has no such ambiguity and was re-validated at
100% (40/40 textures, byte-exact) after the fix.

### 5.3 Pixel formats

| Format | Unity enum | Bytes/px | Notes |
|---|---|---|---|
| Alpha8 | 1 | 1 | Rendered as RGB=0, A=value (matches Unity's own convention) |
| ARGB4444 | 2 | 2 | 4-bit nibbles, ×17 to expand to 8-bit (0xF × 17 = 255) |
| RGB24 | 3 | 3 | |
| RGBA32 | 4 | 4 | |
| DXT1 (BC1) | 10 | block | 4×4 blocks, 8 bytes/block, 2 base colors + 2 interpolated |
| DXT5 (BC3) | 12 | block | 4×4 blocks, 16 bytes/block, separate 3-bit-indexed alpha ramp |

All uncompressed formats read **bottom-up** (Unity's storage convention) and get row-flipped
during decode to produce a normal top-down RGBA buffer for `<canvas>`. DXT1/DXT5 decoding
implements the standard BC1/BC3 block-unpacking algorithm (565 color unpacking, 2-or-4-color
interpolation ramp depending on which reference color is numerically larger, and for DXT5 a
separate 8-value or 6-value alpha ramp selected the same way) with the same bottom-up flip
applied per-block. **Validated pixel-perfect (max channel diff of 1, i.e. float-rounding only)
against `texture2ddecoder`'s reference decode** for both a DXT1 and a DXT5 texture.

### 5.4 Categorization

Not a format detail, but worth documenting: extracted textures sort into **Texture** / **Graphics**
/ **Image** by a simple, transparent heuristic — anything ≤128px on its longest side is
"Graphics" (icons/UI), otherwise DXT-compressed goes to "Image" and uncompressed goes to
"Texture". This is a UX convenience, not a Unity-native concept.

---

## 6. Meshes: a real sequential binary reader

Unlike audio/textures, `Mesh` objects have **variable-length preceding sections** (submesh array,
bind-pose matrices, bone name hashes, bones' bounding boxes — all sized by counts that differ per
mesh), so there's no fixed byte offset to anchor on the way there is for Texture2D. The reader
walks the structure field-by-field instead:

```
name (length-prefixed string, 4-byte aligned)
submesh count (u32) + submeshes[] { firstByte, indexCount, topology, baseVertex,
                                     firstVertex, vertexCount, localAABB(center+extent) }  — 48B each
BlendShapeData: 4 vector counts (shapes/vertices/channels/fullWeights) — must all be 0
                (meshes with actual blend shapes are detected and skipped, not silently misread)
bind pose count (u32) + 4×4 float matrices — 64B each  (this is what flags a mesh as "skinned")
bone name hash count (u32) + u32 hashes
root bone name hash (u32)
bones AABB count (u32) + MinMaxAABB structs — 24B each
mesh compression (u8 + 3 pad) + readable/keepVertices/keepIndices flags (packed u32)
index format (i32: 0=u16, 1=u32)
index buffer: length (u32) + bytes, 4-byte aligned
vertex data: vertexCount (u32) + currentChannels bitmask (u32) + 14× ChannelInfo
             {stream, offset, format, dimension} (4B each) + dataSize length (u32) + bytes
```

Every one of these fields — including the *order* of `vertexCount` vs. the channels bitmask,
which is swapped from an initial (wrong) guess — was validated by re-parsing all 48 meshes in a
real scene and diffing name, index buffer bytes, vertex count, channel array, and the *entire*
vertex data blob against UnityPy field-by-field. 48/48 byte-exact.

### 6.1 Vertex decoding

Each of the (up to) 4 vertex streams has its own **stride**, computed as the sum of
`dimension × formatSize` for every channel assigned to it; stream *offsets* within the data blob
are the cumulative `vertexCount × stride` of prior streams — **no padding/alignment between
streams**, confirmed empirically (an earlier assumption of 16-byte stream alignment was wrong;
the real formula has zero gap). Format codes are the standard `VertexFormat` enum
(`Float32/Float16/UNorm8/SNorm8/UNorm16/SNorm16/UInt8/SInt8/UInt16/SInt16/UInt32/SInt32`).

### 6.2 OBJ export & Three.js preview

Positions/normals/UVs decode from whichever channels are present (index 0/1/4 by Unity
convention) into flat `Float32Array`s, triangle indices get `baseVertex`-offset per submesh, and
the result is either written as a plain-text `.obj` or fed directly into a Three.js
`BufferGeometry` for the in-browser viewer. The viewer uses **hand-rolled orbit controls** (no
`OrbitControls` script dependency) and recomputes the camera's near/far clip planes every frame
relative to the *current* orbit distance (`zoom ± radius×2`) rather than a fixed
`radius×0.001`–`radius×1000` range — the fixed version produced a near:far ratio of ~10,000,000:1,
which exceeds the GPU depth buffer's usable precision and caused foreground/background z-fighting
(surfaces flickering through each other unpredictably). The distance-relative version keeps the
ratio around 9:1 regardless of zoom level.

### 6.3 Character detection (heuristic, explicitly imperfect)

A mesh with `bindPoseCount > 0` is *structurally* skinned to a skeleton — that field is
byte-validated, not guessed. But it turned out to be a **weak signal in practice**: on one real
test project, 288 of 290 meshes had a nonzero bind pose count, including plain primitive props,
making it useless as a "this is a character" filter for that project. It's kept as a hint
(displayed on the card, e.g. "character · 12 bones") rather than a hard category, alongside a
name-substring heuristic (`character|player|hero|enemy|npc|boss|avatar|monster|creature`). A
more reliable signal would require parsing `SkinnedMeshRenderer` vs. `MeshRenderer` objects to
see which meshes are *actually* driven by a skeleton at runtime — not yet implemented.

---

## 7. Multi-file zip orchestration

A `Game_Data` folder has many `.assets` files, each potentially paired with **one or more**
resource files (a single `sharedassetsN.assets` can reference both `sharedassetsN.resource` for
audio *and* `sharedassetsN.assets.resS` for textures simultaneously). Pairing candidates:

1. **Exact name match** (case-insensitive) against the conventional patterns
   (`base.resource`, `base.resS`, `base.assets.resS`, `base_assets.resS`).
2. **Same-directory fallback**: if no exact match, try every resource-like file in the same zip
   folder (handles non-standard/content-addressed naming).
3. **Whole-archive fallback**: if still nothing, try every resource-like file anywhere in the zip.

Tiers 2–3 are safe against false positives specifically *because* every consumer of a resolved
pairing (the AudioClip and Texture2D scanners) independently re-validates that the path string
embedded inside the `.assets` binary matches the candidate file's own name before accepting
anything — a wrong pairing costs some wasted scan time, never incorrect output.

---

## 8. Progress overlay

The full-page progress overlay (percent / last file / ETA / files / size) is driven by **real
numbers**, not a synthetic animation: total byte count is computed upfront from the zip's actual
uncompressed entry sizes, `doneBytes` increments as each `.assets`/resource file is actually
decompressed, and `files`/`size` increment as each output item is actually rendered. ETA is
`remainingBytes / (doneBytes / elapsedSeconds)` — ordinary throughput extrapolation, recomputed
on every update.

---

## 9. Known limitations

- **No TypeTree / no generic object model.** Every format above is a hand-validated heuristic
  scanner. A sufficiently different Unity version *could* shift a struct layout and cause silent
  misses (sanity-bounded reads fail closed, not open — a mismatched layout produces "not found,"
  not garbage output).
- **Compressed texture formats**: only DXT1/DXT5 (BC1/BC3) are decoded. BC4/5/6/7, ETC1/2, ASTC,
  and PVRTC fall back to raw `.bin` export.
- **Blend-shaped meshes** are detected (nonzero shape/vertex/channel counts in `BlendShapeData`)
  and explicitly skipped rather than misread, since the tail struct layout after a nonzero blend
  shape count wasn't reverse-engineered.
- **No AnimationClip support.** Unity's animation format layers compressed keyframe curves and
  (for humanoid rigs) muscle-space retargeting curves on top of generic property bindings — a
  meaningfully different and larger reverse-engineering task than anything above, deliberately
  scoped out rather than shipped unvalidated.
- **No Material/Renderer parsing**, so texture↔mesh associations for a specific character model
  are name-heuristic only, not ground truth.

## 10. On Unity's own branded assets

When run against Unity's `unity default resources` file (bundled with every Editor install),
the scanner also finds Unity Technologies' own splash-screen and watermark artwork
(`UnitySplash-cube`, `UnityWatermark-*`) — these are copyrighted/trademarked brand assets, not
generic engine UI, so they're excluded by default via a name-based skip list. An opt-in toggle
exists in the code (checkbox, hidden from the UI by default) rather than being surfaced as a
public feature.
