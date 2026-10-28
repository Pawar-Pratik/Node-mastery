# Lesson 12 — Buffers, bytes & encodings

> **Why this lesson exists:** HTTP bodies, files and sockets carry bytes, not strings, and the bugs at that boundary are silent: a `₹` split across two chunks turns into garbage, a `Content-Length` computed from `string.length` truncates a response, a webhook signature computed over re-serialized JSON never matches, and a 16-byte slice quietly pins 64 KiB of memory. This lesson is the byte-level model you need to handle binary data, raw bodies and encodings without corrupting anything, and it's the foundation for streams in [Lesson 13](13-streams-and-backpressure.md).

**Time:** ~65 minutes · **Prereq:** Lesson 02

---

## 1. The idea in one sentence

> **Everything that crosses a process boundary is bytes, and a string is only one interpretation of them: decode once, at the edge, after you have all the bytes, and never use a string's length to measure its size.**

---

## 2. Why it exists: JavaScript had no bytes

JavaScript strings are sequences of UTF-16 code units, immutable and managed by V8. When Ryan Dahl built Node in 2009, the language had no way to hold raw bytes at all (typed arrays came later), yet a server's entire job is reading and writing bytes. Node invented `Buffer` for that, and later rebuilt it on top of the standard `Uint8Array`.

The boundary between the two worlds is where data gets corrupted, usually without an error:

| Bug | What you see |
|---|---|
| Decoding each network chunk separately | `�` in names, amounts and notes, intermittently, only for non-ASCII text |
| `Content-Length: body.length` | Responses cut short whenever the body contains `₹` or an emoji |
| HMAC over `JSON.stringify(req.body)` | Every webhook signature check fails, or passes only for some senders |
| `Buffer.allocUnsafe` sent without filling it | Fragments of other requests' data in a response |
| Caching `chunk.subarray(0, 16)` | Memory grows 4,000× faster than the data you think you're keeping |

---

## 3. What a Buffer is

**`Buffer` is a subclass of `Uint8Array`.** Anything that accepts a `Uint8Array` accepts a Buffer; Buffer adds encodings (`toString("base64")`), binary read/write helpers, `equals`, `compare` and a few allocation strategies.

Its memory is **not** on the V8 heap:

```text
 V8 heap                               Outside the heap (native memory)
┌──────────────────────┐              ┌──────────────────────────────────┐
│ Buffer object (tiny) │──────────────▶│ ArrayBuffer backing store         │
│ { buffer, byteOffset,│              │ [e2 82 b9 35 30 30 ...] 64 KiB    │
│   length }           │      ┌──────▶│                                   │
├──────────────────────┤      │       └──────────────────────────────────┘
│ subarray(0,16) (tiny)│──────┘        counted in memoryUsage().external
└──────────────────────┘               and .arrayBuffers, not heapUsed
```

Allocate a 50 MB Buffer and `process.memoryUsage()` shows `external` and `arrayBuffers` up by 50 MB while `heapUsed` barely moves. Three consequences:

- **`--max-old-space-size` doesn't limit Buffers.** A service can be OOMKilled by the kernel while its heap looks small ([L27](../07-production/27-memory-and-gc.md)).
- **The backing store is freed only when every JavaScript object pointing at it is garbage-collected.** That's what makes the retention leaks in §4 possible.
- **A view is not a copy.** `subarray` makes a new tiny object pointing into the *same* backing store.

### `subarray` shares; `slice` is deprecated because it lies

```ts
const buf = Buffer.from([1, 2, 3]);
const view = buf.subarray(0, 1);
view[0] = 9;
console.log(buf[0]);                  // 9: same memory

new Uint8Array([1, 2, 3]).slice(0, 1); // Uint8Array#slice COPIES
Buffer.from([1, 2, 3]).slice(0, 1);    // Buffer#slice SHARES (deprecated: same name, opposite behaviour)
```

`Buffer.prototype.slice` predates typed arrays and shares memory, while the standard `Uint8Array.prototype.slice` copies. Same name, opposite semantics, so Node deprecated Buffer's version: **use `subarray` when you want a view and `Buffer.from(view)` when you want a copy**, and the code says which you meant.

---

## 4. Allocation: `alloc`, `allocUnsafe` and the pool

| Call | Memory | Use it when |
|---|---|---|
| `Buffer.alloc(n)` | Zero-filled | The default |
| `Buffer.allocUnsafe(n)` | **Uninitialized**, may come from the shared pool | You will overwrite every byte immediately |
| `Buffer.allocUnsafeSlow(n)` | Uninitialized, never pooled | A long-lived small buffer (below) |
| `Buffer.from(string, enc)` | A copy of the encoded string | Text to bytes |
| `Buffer.from(buffer)` / `Buffer.from(array)` | A copy | Detaching from a bigger backing store |
| `Buffer.from(arrayBuffer, offset, length)` | **Shares** the ArrayBuffer | Zero-copy wrapping |

**"Uninitialized" means whatever was in that memory before**: freed buffers that held request bodies, tokens or passwords. Send an `allocUnsafe` buffer without filling it and you can leak another request's data. (The old `new Buffer(n)` constructor returned uninitialized memory too, which is a large part of why it's deprecated.) `allocUnsafe` is only faster because it skips the zero-fill; use it only where every byte gets written, like this length-prefixed frame:

```ts
export function encodeFrame(payload: Buffer): Buffer {
  const out = Buffer.allocUnsafe(4 + payload.length);   // ✅ every byte overwritten below
  out.writeUInt32BE(payload.length, 0);
  payload.copy(out, 4);
  return out;
}
```

`--zero-fill-buffers` forces zero-filling everywhere, at a throughput cost, if you can't audit a dependency.

### The pool

Small allocations would be slow if each one went to the system allocator, so Node keeps a shared **8 KiB pool** (`Buffer.poolSize` = 8192). `allocUnsafe`, `Buffer.from(string)` and friends carve requests **smaller than 4 KiB** (`poolSize >>> 1`) out of the current pool; larger ones get their own allocation. You can see it: `Buffer.from("hi").buffer.byteLength` is 8192.

### Two retention leaks

**Leak 1: a small view of a big chunk.** Network and file reads arrive in chunks of tens of KiB (a file read stream reads 64 KiB at a time by default). Keep a few bytes as a view and you keep the whole chunk:

```ts
// ❌ Caching the 16-byte delivery id as a view: each entry pins its 64 KiB chunk
idCache.set(key, chunk.subarray(offset, offset + 16));

// ✅ Copy what you keep; the chunk can be freed
idCache.set(key, Buffer.from(chunk.subarray(offset, offset + 16)));
```

10,000 cached ids: about 160 KB of data, but up to 640 MB of pinned chunks with the view.

**Leak 2: a long-lived pooled buffer pins its pool slab.** The copy above is under 4 KiB, so it lands in the shared 8 KiB pool, and that slab can't be freed while *any* slice of it is alive. Keep one small buffer from each of thousands of slabs (cache entries created over hours, interleaved with other allocations) and you retain up to 8 KiB per entry. **For small buffers you'll keep indefinitely, Node's docs recommend `allocUnsafeSlow` + copy**, which gives an exact-size, unpooled allocation:

```ts
export function retainCopy(view: Buffer): Buffer {
  const own = Buffer.allocUnsafeSlow(view.length);   // its own backing store, not a pool slab
  view.copy(own);                                     // every byte overwritten: unsafe is fine
  return own;
}
```

Often the simplest fix is not to keep bytes at all: `view.toString("hex")` gives you a string on the V8 heap with no link to any backing store.

---

## 5. Encodings

| Encoding | Bytes per character | Notes |
|---|---|---|
| `utf8` (default) | 1–4 | **Decoding invalid sequences produces `U+FFFD` (`�`) silently: lossy** |
| `utf16le` | 2 or 4 | V8's internal representation; rare on the wire |
| `latin1` (alias `binary`) | 1 | Maps bytes 0–255 to code points 0–255: the only lossless bytes → string → bytes round trip |
| `ascii` | 1 | Strips the high bit. Avoid |
| `base64` | 4 chars per 3 bytes | **The decoder is lenient**: it skips characters outside the alphabet instead of failing |
| `base64url` | as base64 | URL-safe alphabet (`-`, `_`), no padding: tokens, JWTs, cursors |
| `hex` | 2 chars per byte | **The decoder stops at the first invalid pair**: `Buffer.from("abz1", "hex")` is one byte, and an odd length drops the last digit |

Two rules follow:

- **Never round-trip bytes you need exactly through `utf8`.** A signature, a hash or a binary file that contains an invalid UTF-8 sequence comes back different. If you must carry bytes in a string, use `base64` or `hex`.
- **Validate encoded input yourself.** Node won't throw on malformed `hex` or `base64`; it quietly decodes less. Check the format with a strict regex and the decoded length before trusting it (Ledger's signature parser in §11 does both).

---

## 6. `string.length` is not a size

`length` counts UTF-16 code units, not bytes and not characters:

```ts
const note = "Refund ₹500 \u{1F44D}";   // "Refund ₹500 " plus a thumbs-up emoji

note.length;                    // 14: "₹" is 1 code unit; the emoji is 2 (a surrogate pair)
Buffer.byteLength(note);        // 18: "₹" is 3 bytes (e2 82 b9); the emoji is 4
```

### The `Content-Length` bug

```ts
// ❌ Declares 14 bytes, sends 18
res.setHeader("Content-Length", note.length);
res.end(note);

// ✅
res.setHeader("Content-Length", Buffer.byteLength(note));
res.end(note);
```

The client reads exactly 14 bytes and stops: it receives `"Refund ₹500 "` and the emoji is cut off. The 4 extra bytes are still on the connection, and on a keep-alive connection the client may parse them as the start of the *next* response. Node doesn't stop you by default (there's an opt-in `res.strictContentLength` that throws on a mismatch). The simplest fix is not to set it at all: `res.end(string)` and Express's `res.send` compute the byte length for you.

The same confusion bites wherever a limit is in bytes: Ledger stores `webhook_deliveries.response_body` truncated to 2 KB. Truncate the **bytes**, and don't cut a character in half:

```ts
import { StringDecoder } from "node:string_decoder";

export function truncateUtf8(bytes: Buffer, maxBytes: number): string {
  // write() returns only complete characters and holds back a split one at the end
  return new StringDecoder("utf8").write(bytes.subarray(0, maxBytes));
}
```

---

## 7. Characters split across chunks

TCP and files deliver bytes in chunks whose boundaries have nothing to do with characters. A 3-byte `₹` can arrive as 1 byte at the end of one chunk and 2 at the start of the next:

```ts
const rupee = Buffer.from("₹");                          // <e2 82 b9>
rupee.subarray(0, 1).toString() + rupee.subarray(1).toString(); // "���": three replacement chars
```

Each `toString()` sees an incomplete sequence and replaces it. The fix is a decoder that **remembers the incomplete tail between calls**:

```ts
// ❌ Mojibake whenever a multi-byte character straddles a chunk boundary
let text = "";
for await (const chunk of stream) text += chunk.toString("utf8");

// ✅ StringDecoder carries partial characters into the next write()
import { StringDecoder } from "node:string_decoder";
const decoder = new StringDecoder("utf8");
let text = "";
for await (const chunk of stream) text += decoder.write(chunk);
text += decoder.end();                       // flush anything left (an incomplete tail becomes �)

// ✅ The web-standard equivalent
const td = new TextDecoder("utf-8");
for await (const chunk of stream) text += td.decode(chunk, { stream: true });
text += td.decode();
```

`readable.setEncoding("utf8")` does the same thing inside the stream (it uses a StringDecoder), so `'data'` delivers correctly decoded strings. Or simplest of all: collect the bytes (§9) and decode once at the end. Splitting the decoded text into *lines* has its own chunk-boundary problem, which is [L14](14-composing-streams.md)'s.

This bug hides in testing because ASCII is one byte per character and never splits. **It appears in production the first time a merchant's customer has a name in Devanagari or a refund note with `₹`, and only when the split lands mid-character.** Test with non-ASCII data at chunk boundaries deliberately.

---

## 8. Binary layouts: endianness and framing

Binary protocols put integers at fixed offsets. The byte order is part of the format:

```ts
const buf = Buffer.alloc(8);
buf.writeUInt32BE(4999, 0);          // big-endian: 00 00 13 87   (network byte order)
buf.writeUInt32LE(4999, 4);          // little-endian: 87 13 00 00 (x86-native formats)
buf.readUInt32BE(0);                 // 4999
buf.readBigUInt64BE(0);              // a bigint: 64-bit values exceed Number's 2^53 (L02)
```

Big-endian is the convention for network protocols; the Postgres wire protocol, for example, uses big-endian lengths. Read with the wrong endianness and 4999 becomes 2,266,169,344, with no error. The `encodeFrame` in §4 is the sending half of **length-prefixed framing**: a 4-byte big-endian length, then the payload. Decoding it from a socket, where frames split and coalesce across `'data'` events, is in [L16](../04-networking/16-tcp-and-net.md).

---

## 9. Collecting chunks: concat cost and size caps

```ts
// ❌ O(n²): every chunk copies everything collected so far
let body = Buffer.alloc(0);
for await (const chunk of req) body = Buffer.concat([body, chunk]);

// ❌ Decodes each chunk separately (the §7 bug), and no size limit at all
let text = "";
for await (const chunk of req) text += chunk;
```

The correct shape: keep the chunks in an array, count bytes as they arrive, **stop the moment the cap is exceeded**, and concatenate once with the known total:

```ts
export class PayloadTooLargeError extends Error {
  readonly code = "payload_too_large";
  readonly limit: number;
  constructor(limit: number) {
    super(`body exceeds ${limit} bytes`);
    this.name = "PayloadTooLargeError";
    this.limit = limit;
  }
}

export async function collectBody(source: AsyncIterable<Buffer | string>, limit: number): Promise<Buffer> {
  const chunks: Buffer[] = [];
  let total = 0;
  for await (const chunk of source) {
    const buf = typeof chunk === "string" ? Buffer.from(chunk) : chunk;
    total += buf.length;
    if (total > limit) throw new PayloadTooLargeError(limit);   // leaving for-await destroys the stream
    chunks.push(buf);
  }
  return Buffer.concat(chunks, total);                            // one allocation, one copy
}
```

Passing `total` to `Buffer.concat` saves it a pass over the chunks to compute the length. Throwing out of `for await` calls the stream's `return()`, which destroys it, so the rest of an oversized upload is never buffered. For an HTTP request, destroying the request also kills the socket before you can answer; [L17](../04-networking/17-http-from-scratch.md) shows how to reject with a proper 413 (check `Content-Length` up front, then stop reading). The policy side of body limits is in [API L16](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/16-owasp-and-hardening.md).

---

## 10. Comparing secrets: `timingSafeEqual`

`a.equals(b)`, `===` on strings and a hand-written loop all stop at the first differing byte, so how long the comparison takes reveals how many leading bytes matched. Over enough requests an attacker can recover a valid signature byte by byte. The attack is covered in [API L12](https://github.com/Pawar-Pratik/API-mastery/blob/main/03-security/12-authentication-landscape.md); the Node mechanics are here.

**`crypto.timingSafeEqual(a, b)` takes time that depends only on the length, and it throws `RangeError` (`ERR_CRYPTO_TIMING_SAFE_EQUAL_LENGTH`) if the lengths differ.** So:

- **Compare fixed-length digests**, never raw secrets of variable length: HMAC-SHA256 output is always 32 bytes.
- **Check the provided value's format and length first.** Revealing that a signature has the wrong length leaks nothing: everyone knows a SHA-256 digest is 32 bytes.
- **Decode with a strict format check.** Because `Buffer.from(sig, "hex")` silently truncates at the first bad character, a malformed signature would otherwise reach `timingSafeEqual` with the wrong length and throw, turning a bad request into a 500.

---

## 11. Ledger: the webhook HMAC over raw bytes

Ledger signs every webhook with HMAC-SHA256 over `timestamp.rawBody` ([API L23](https://github.com/Pawar-Pratik/API-mastery/blob/main/05-beyond-rest/23-realtime-and-webhooks.md) has the design: timestamps, rotation, replay window). The Node side has one rule that everything else follows from: **the signature covers the exact bytes on the wire.**

Why parsed-and-re-serialized JSON can't be verified:

```text
bytes the sender signed:   {"amount_minor": 4999, "currency": "inr", "note": "₹", "10": 1, "b": 2}
JSON.stringify(parse(…)):  {"10":1,"amount_minor":4999,"currency":"inr","note":"₹","b":2}
                             ▲ integer-like keys move first   ▲ escapes decoded   ▲ whitespace gone
```

Whitespace disappears (and many senders' serializers add it), `₹` escapes become the literal character, numbers are reformatted (`1.0` becomes `1`), integer-like keys are reordered ahead of the others, and big integers lose precision. Any one of those changes the bytes, and **a one-byte difference is a completely different HMAC.**

### `hmac.ts`

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

const TOLERANCE_S = 300;                 // ±5 minutes: the replay window
const SIG_HEX = /^[0-9a-f]{64}$/;        // SHA-256 = 32 bytes = exactly 64 hex characters

function digest(secret: string, timestamp: number, rawBody: Buffer): Buffer {
  // Two update() calls equal one HMAC over the concatenation, without building a joined Buffer
  return createHmac("sha256", secret).update(`${timestamp}.`).update(rawBody).digest();
}

/** One v1 per live secret, so receivers keep verifying during a rotation. */
export function sign(rawBody: Buffer, secrets: readonly string[], timestamp = Math.floor(Date.now() / 1000)): string {
  return [`t=${timestamp}`, ...secrets.map((s) => `v1=${digest(s, timestamp, rawBody).toString("hex")}`)].join(",");
}

export type Verified = { ok: true; timestamp: number } | { ok: false; reason: "malformed" | "stale" | "mismatch" };

export function verify(header: string, rawBody: Buffer, secrets: readonly string[], nowMs = Date.now()): Verified {
  let timestamp = Number.NaN;
  const provided: Buffer[] = [];
  for (const part of header.split(",")) {
    const eq = part.indexOf("=");
    const key = part.slice(0, eq).trim();
    const value = part.slice(eq + 1).trim();
    if (key === "t" && /^\d{1,12}$/.test(value)) timestamp = Number(value);
    else if (key === "v1" && SIG_HEX.test(value)) provided.push(Buffer.from(value, "hex"));   // exactly 32 bytes
  }
  if (!Number.isFinite(timestamp) || provided.length === 0) return { ok: false, reason: "malformed" };
  if (Math.abs(nowMs / 1000 - timestamp) > TOLERANCE_S) return { ok: false, reason: "stale" };

  for (const secret of secrets) {                         // current and previous during rotation
    const expected = digest(secret, timestamp, rawBody);
    for (const sig of provided) {
      if (timingSafeEqual(expected, sig)) return { ok: true, timestamp };   // equal lengths guaranteed
    }
  }
  return { ok: false, reason: "mismatch" };
}
```

The sender's discipline is the mirror image: **serialize once to a Buffer, sign that Buffer, send that Buffer** with `Content-Length: buf.length`. Never `JSON.stringify` a second time for sending.

### Capturing the raw body

The receiver's job is to get the bytes *before* anything parses them, with a size cap:

```ts
// Express, on the webhook route only: req.body is a Buffer
app.post("/webhooks/ledger", express.raw({ type: "application/json", limit: "1mb" }), (req, res) => {
  const result = verify(req.header("ledger-signature") ?? "", req.body as Buffer, [CURRENT, PREVIOUS]);
  if (!result.ok) return res.status(400).json({ code: "invalid_signature", reason: result.reason });
  const event = JSON.parse((req.body as Buffer).toString("utf8"));   // parse AFTER verifying
  // …ack fast, process async (API L23)
});

// When a route needs both the parsed JSON and the bytes: body-parser's verify hook sees the raw Buffer
app.use(express.json({ limit: "1mb", verify: (req, _res, buf) => { (req as RawBodyRequest).rawBody = buf; } }));

// Framework-free (the SDK's helper, L17's server): collectBody(req, 1_048_576)
```

A detail worth knowing: the API track's receiver converted the Buffer to a UTF-8 string and signed the string. That works for valid UTF-8, but a body containing an invalid sequence decodes to `�` and the HMAC over the string no longer matches the bytes. **Sign and verify the Buffer; decode only to parse.**

---

## 12. Production rules

| Rule | Why |
|---|---|
| **Keep data as bytes until you have all of it, then decode once** | Per-chunk decoding corrupts characters split across chunks |
| **Decode streamed text with `StringDecoder`, `TextDecoder({ stream: true })` or `setEncoding`** | They carry incomplete multi-byte sequences across chunks |
| **Measure with `Buffer.byteLength`, never `string.length`** | `length` counts UTF-16 code units; headers and limits are in bytes |
| **Let `res.end(string)` / `res.send` compute `Content-Length`, or use `byteLength`** | A short `Content-Length` truncates the body and can corrupt the next response on the connection |
| **`Buffer.alloc` by default; `allocUnsafe` only when every byte is overwritten** | Uninitialized memory can contain other requests' data |
| **Copy small views you keep (`Buffer.from(view)`, or `allocUnsafeSlow` + `copy` for long-lived ones)** | A view pins its whole chunk; a pooled copy pins an 8 KiB slab |
| **Use `subarray` for views; never `Buffer#slice`** | `slice` shares memory despite `Uint8Array#slice` copying; deprecated for that reason |
| **Collect chunks in an array with a byte cap; `Buffer.concat(chunks, total)` once** | Concat-per-chunk is O(n²); an uncapped body is a memory DoS |
| **Validate `hex`/`base64` input format and decoded length yourself** | Node's decoders silently truncate or skip invalid characters |
| **Never round-trip exact bytes through `utf8` strings** | Invalid sequences become `U+FFFD`; the bytes change |
| **Compare secrets with `timingSafeEqual` on fixed-length digests, after a length check** | Early-exit comparison leaks timing; unequal lengths throw |
| **HMAC over the raw body Buffer, captured before parsing, verified before parsing** | Re-serialized JSON differs in whitespace, escapes, numbers and key order |
| **Remember Buffers live outside the heap** | Heap limits don't cap them; watch `external`/`arrayBuffers` and RSS |

> **Spring equivalent:** the same raw-body problem exists in Spring: a webhook endpoint takes `@RequestBody byte[]` instead of a DTO, or a filter caches the body with `ContentCachingRequestWrapper`, which buffers the whole body in memory and so needs a size cap exactly like `collectBody` ([SpringBoot L17](https://github.com/Pawar-Pratik/SpringBoot/blob/main/05-web/17-openapi-cors-filters-interceptors.md)). And Java's `String.length()` also counts UTF-16 code units, so the `Content-Length` bug is portable: the fix there is `getBytes(StandardCharsets.UTF_8).length`.

---

## 13. Interview traps

**Q1. "Why must a webhook HMAC be verified over the raw body?"**
Because the signature covers the exact bytes the sender transmitted, and parsing then re-serializing changes them: whitespace is dropped, `\u` escapes are decoded, numbers are reformatted and integer-like keys are reordered. Any single byte difference gives a different HMAC. So you capture the body as a Buffer before any JSON parser runs (`express.raw` on that route, or the `verify` hook), verify, then parse.
**Follow-up:** "Can you verify over `rawBody.toString("utf8")` instead?" Only if the body is valid UTF-8; invalid sequences decode to `U+FFFD` and the HMAC changes. Verify the Buffer.

**Q2. "`Buffer.alloc` vs `Buffer.allocUnsafe`?"**
`alloc` zero-fills; `allocUnsafe` returns uninitialized memory, possibly a slice of the shared 8 KiB pool, which can still contain data from freed buffers such as other requests' bodies. `allocUnsafe` is faster only by the cost of the fill, so it's correct only when you immediately overwrite every byte. Sending an unfilled one is a data leak.

**Q3. "Why can `chunk.toString()` corrupt text?"**
Chunk boundaries fall at arbitrary bytes, so a multi-byte UTF-8 character can be split across two chunks. Decoding each chunk alone turns both halves into replacement characters. Use `StringDecoder` or `TextDecoder` with `{ stream: true }` (or `setEncoding`), which hold an incomplete tail until the next chunk, or collect bytes and decode once.

**Q4. "`string.length` vs byte length?"**
`length` is the number of UTF-16 code units: `"₹".length` is 1 but it's 3 bytes in UTF-8, and an emoji is length 2 and 4 bytes. Anything measured in bytes (`Content-Length`, column or payload limits, truncation) must use `Buffer.byteLength`. Getting it wrong truncates responses only when the text isn't ASCII, which is why it survives testing.

**Q5. "Why `timingSafeEqual`, and why does it need equal lengths?"**
Ordinary comparisons return at the first mismatched byte, so response timing leaks how much of a guessed signature was right. `timingSafeEqual` runs in time dependent only on length, and throws on unequal lengths because it can't compare them in constant time. So you compare fixed-length digests and check the length first, which leaks nothing since digest sizes are public.

**Q6. "Where does Buffer memory live?"**
Outside the V8 heap, in a native backing store; the JavaScript object is a small handle. It shows up in `memoryUsage().external` and `arrayBuffers`, not `heapUsed`, isn't limited by `--max-old-space-size`, and is freed only when every object referencing it is collected.
**Follow-up:** "So how can a pod get OOMKilled with a 200 MB heap?" Off-heap memory like Buffers counts toward RSS and the container limit but not the heap limit ([L27](../07-production/27-memory-and-gc.md)).

**Q7. "`subarray`, `slice` and `Buffer.from(buf)`: what's the difference?"**
`subarray` returns a view sharing memory. `Buffer#slice` also shares, which contradicts `Uint8Array#slice` (a copy), so it's deprecated. `Buffer.from(buf)` copies. The practical consequence: keeping a small view keeps its entire parent alive, so copy what you retain.

**Q8. "What does `Buffer.from("abz1", "hex")` return?"**
One byte, `0xab`: the hex decoder stops at the first invalid pair and odd trailing digits are dropped, with no error. Base64 decoding is similarly lenient. That's why signature parsing validates the format with a strict regex and checks the decoded length before comparing.

**Q9. "Why is `Buffer.concat` in a loop slow?"**
Each call allocates a new buffer and copies everything collected so far, so collecting n chunks copies O(n²) bytes. Push chunks into an array, track the total, enforce a cap as you go, and call `Buffer.concat(chunks, total)` once.

**Q10. "What does Node's Buffer pool buy you, and what does it cost?"**
Allocations under 4 KiB are carved out of a shared 8 KiB slab, avoiding a system allocation per small buffer. The cost is retention: a slab is freed only when every slice of it is unreachable, so long-lived small buffers can each pin 8 KiB. For small buffers you keep indefinitely, copy into `allocUnsafeSlow` (or convert to a string).

---

## 14. Build & break

### Build — `src/lib/hmac.ts`
- `sign(rawBody, secrets, timestamp?)` and `verify(header, rawBody, secrets, nowMs?)` from §11, Buffers in and out, no string round trips.
- Tests: sign-then-verify succeeds; flipping any single byte of the body fails with `mismatch`; a timestamp 301 s old fails with `stale`; a `v1` with 63 hex characters, a non-hex character, or a missing `t` returns `malformed` without throwing; a header signed with the previous secret verifies when both secrets are configured; a body containing an invalid UTF-8 byte (`Buffer.from([0x7b, 0xff, 0x7d])`) verifies over the Buffer.
- Use it in two places: the webhook sender (serialize once, sign the Buffer, send the Buffer) and the `@ledger/node` SDK's `webhooks.verify()` ([L32](../08-shipping/32-packaging-and-publishing.md)).

### Build — `src/lib/body.ts`
- `collectBody(source, limit)` and `PayloadTooLargeError` from §9, plus `truncateUtf8(bytes, maxBytes)` from §6 for `webhook_deliveries.response_body`.
- Tests: a body exactly at the limit succeeds; one byte over throws and leaves the source stream `destroyed`; a 3-byte `₹` split across two chunks comes back intact; truncating `"ab₹"` to 4 bytes returns `"ab"`, not `"ab�"`.

### Break — five experiments
1. **Re-serialized JSON.** Sign `Buffer.from('{"amount_minor": 4999, "currency": "inr"}')` (note the spaces, as Python's `json.dumps` produces by default). Verify against `Buffer.from(JSON.stringify(JSON.parse(body.toString())))`: `mismatch`. Add `"10": 1` after `"currency"` and print the re-serialized string to watch the key jump to the front. Verify the original Buffer: `ok`.
2. **The split rupee.** Split `Buffer.from("Refund ₹500")` at byte 8 (inside the `₹`) into two chunks, feed them to `Readable.from([a, b])`, and build the text with `chunk.toString()` per chunk: you get `�` characters. Rerun with `StringDecoder`, then with `stream.setEncoding("utf8")`: correct both times.
3. **The short `Content-Length`.** Serve `"Refund ₹500 \u{1F44D}"` with `Content-Length` set to `note.length`, then `curl.exe -v http://localhost:3000/note` (use `curl.exe`: in PowerShell, `curl` is an alias). The body arrives cut short, missing the emoji. Switch to `Buffer.byteLength(note)` (or drop the header) and the full body arrives.
4. **Uninitialized memory.** In a loop, create a few thousand `Buffer.from("SECRET-TOKEN-" + i)` strings of a few KiB each and drop them, then print `Buffer.allocUnsafe(64 * 1024).toString("latin1")` and search it for `SECRET`. Whether you see fragments depends on the allocator and timing, so run it a few times: the point is that **nothing guarantees zeros**. Then `Buffer.alloc`: zeros, always.
5. **The pinned chunks.** In a loop of 2,000: allocate a 64 KiB chunk, make one throwaway small allocation (`Buffer.from("x".repeat(3000))`, which comes from the pool), and keep a 16-byte `subarray` of the chunk in an array. Run with `--expose-gc`, call `global.gc()` a few times, and print `process.memoryUsage().arrayBuffers`: about 128 MB retained for 32 KB of data. Run it again (fresh process) keeping `Buffer.from(view)`: it drops to a few MB, because each small copy now pins an 8 KiB pool slab it shares with the throwaway allocations. Run once more with `retainCopy` (`allocUnsafeSlow`) or `view.toString("hex")`: back to roughly the baseline.

### Explain out loud (90 seconds)
1. What a Buffer is and where its memory lives, and why that matters for container limits.
2. `alloc` vs `allocUnsafe`, the 8 KiB pool, and the two retention leaks.
3. `string.length` vs `Buffer.byteLength`, with the `Content-Length` bug.
4. Why per-chunk decoding corrupts text, and the decoders that fix it.
5. Why Ledger's webhook HMAC is computed and verified over raw bytes, and why `timingSafeEqual` needs equal-length digests.

---

## What's next

You can now hold bytes correctly. The next question is how to move an unbounded amount of them, a 4-million-row export or a 2 GB upload, through a process with bounded memory. That's streams, and the contract that makes them work: backpressure, the single most important idea in this module.

Next → **[Lesson 13: Streams & backpressure](13-streams-and-backpressure.md)**
