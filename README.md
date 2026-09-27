# Wastdlib

**CPython's C standard-library modules — unmodified — compiled to WebAssembly
and running inside [Brython](https://brython.info), on the
[Wasthon](https://github.com/fgallaire/wasthon) C-API bridge.**

Wastdlib is the modules; Wasthon is the bridge they are compiled against
(`build.sh` takes it from a Wasthon checkout, `WASTHON_DIR=`, or clones
`wasthon@main` into `.wasthon/`). The same bridge carries NumBry (NumPy,
SciPy, pandas…), BryTorch (PyTorch) and Brygame (pygame).

## What it does

Brython ships a Python 3 runtime in the browser by compiling Python to
JavaScript. The catch: CPython's stdlib modules written in C (`_sha256`,
`zlib`, `_sre`, `_decimal`, …) don't compile to JS — Brython has either
re-implemented them in JavaScript / pure Python (smaller surface, sometimes
worse perf, the occasional silent semantic divergence) or simply doesn't
ship them at all.

Wastdlib takes the **unmodified C source** of those CPython modules,
compiles them to WebAssembly with Emscripten, and exposes them to Brython
through a minimal CPython C-API bridge. Result for Brython users:

- The same `import hashlib`, `import struct`, `import zlib`, `import
  _decimal` works — and flies. Bulk-operation speedups range from 4× to
  **131×** depending on module.
- **New algorithms Brython doesn't ship**: full SHA-3, BLAKE2, HMAC, full
  Unicode database, `_bz2`, `_lzma`, `_zstd`, real `array.array`.
- **Modules that now work**: Brython's `_struct`, `unicodedata`, `zlib`,
  `array` are partially broken / very lossy / unusable in some cases —
  Wastdlib implementations are bit-exact with CPython.
- **Bit-exact CPython semantics** — when CPython fixes a bug or adds a
  feature in 3.x, Wastdlib picks it up by recompiling, not re-porting.

## Status

### Headline result — `_json.encode_basestring`: 25× to 332× faster

| Input                  | Wastdlib  | Brython pure-Python | Speedup    |
| ---------------------- | -------- | ------------------- | ---------- |
| 11-char ASCII          | 0.014 ms | 0.358 ms            | **25.57×** |
| 1300-char ASCII        | 0.110 ms | 22.88 ms            | **208×**   |
| 1700-char w/ escapes   | 0.090 ms | 24.12 ms            | **268×**   |
| 11000-char Lorem Ipsum | 0.450 ms | 149.45 ms           | **332×**   |

`_json.encode_basestring` produces ~24 MB/s of JSON encoding at 11 KB
input — basically native libC speed, against a pure-Python loop saturated
at ~75 KB/s. Largest speedup we've measured on any ported module.

### Modules ported

| Module         | What it provides                                         | `.wasm`     |
| -------------- | -------------------------------------------------------- | ----------- |
| `_md5`         | md5                                                      | 14 KB       |
| `_sha1`        | sha1                                                     | 12 KB       |
| `_sha2`        | sha224, sha256, sha384, sha512                           | 36 KB       |
| `_sha3`        | sha3_224, sha3_256, sha3_384, sha3_512                   | 26 KB       |
| `_blake2`      | blake2b, blake2s                                         | 36 KB       |
| `_hmac`        | HMAC over any of the hashes above                        | 98 KB       |
| `_zlib`        | compress / decompress, crc32, adler32                    | 83 KB       |
| `_bz2`         | bzip2 compress / decompress (libbz2 1.0.8)               | 91 KB       |
| `_lzma`        | XZ / LZMA / raw (xz-utils 5.4.6, pre-CVE-2024-3094)      | 133 KB      |
| `_zstd`        | Zstandard compress / decompress (libzstd 1.5.6)          | 555 KB      |
| `_sre`         | regex engine                                             | 73 KB       |
| `_random`      | Mersenne Twister (bit-exact with CPython)                | 12 KB       |
| `_struct`      | binary pack/unpack, full format-code coverage            | 29 KB       |
| `_decimal`     | arbitrary-precision decimal arithmetic (libmpdec)        | 360 KB      |
| `_csv`         | C-level CSV reader/writer state machine                  | 25 KB       |
| `array`        | typed arrays (b/B/h/H/i/I/l/L/q/Q/f/d)                   | 41 KB       |
| `pyexpat`      | XML parser (libexpat 2.8.2)                              | 169 KB      |
| `_json`        | JSON encoder/decoder C accelerator                       | 28 KB       |
| `math`         | int-heavy fns (factorial, gcd, isqrt, prod, …)           | 77 KB       |
| `cmath`        | complex math (sqrt/exp/log/sin/polar/rect/…)             | 51 KB       |
| `unicodedata`  | full Unicode 15.x database + normalization               | 669 KB      |
| `_statistics`  | `_normal_dist_inv_cdf` (Wichura AS241)                   | 15 KB       |
| `_pickle`      | C accelerator for pickle (protocols 0-5)                 | 79 KB       |
| `_sqlite3`     | SQLite 3.46.1 embedded DB (FTS5 + RTREE + JSON1)         | 730 KB      |
| `binascii`     | hex / base64 / CRC encoding (CPython C accelerator)      | 24 KB       |
|                | **Total**                                                | **~3.4 MB** |

### Highlight benchmarks — Wastdlib vs Brython

**hashlib** (0.5 MB payload, MB/s):

| Algo   | Wastdlib | Brython | Speedup   |
| ------ | ------- | ------- | --------- |
| md5    | 238     | 17.7    | **13.5×** |
| sha256 | 119     | 20.6    | 5.8×      |
| sha512 | 192     | 8.3     | **23.3×** |
| sha3_* | 51-82   | absent  | ∞         |

**`_struct`** (per-call ms, 100 ops/loop):

| Operation   | Wastdlib | Brython | Speedup  |
| ----------- | ------- | ------- | -------- |
| pack '>I'   | 1.17    | 22.4    | **19×**  |
| pack '<10I' | 1.53    | 201     | **131×** |
| pack '>d'   | 0.73    | 34.0    | **46×**  |

**`_decimal`** (ms/op vs Brython's `_pydecimal`):

| Operation                         | Wastdlib | _pydecimal | Speedup |
| --------------------------------- | ------- | ---------- | ------- |
| Decimal('3.14') × Decimal('2.71') | 0.04    | 1.8        | **46×** |
| 1000 mul chain                    | 1.7     | 31         | **18×** |

**`_csv`** (parse/write, ms/op):

| Operation                     | Wastdlib | csv.py | Speedup   |
| ----------------------------- | ------- | ------ | --------- |
| parse 1000 simple rows        | 20.2    | 495.6  | **24.5×** |
| parse 500 quoted-comma rows   | 15.9    | 314.7  | **19.8×** |
| parse tab-delimited 1000 rows | 71.0    | 630.5  | **8.9×**  |

**Compression family** (MB/s throughput; higher is better):

| Op                      | _bz2 | _lzma | _zstd |
| ----------------------- | ---- | ----- | ----- |
| compress 10 KB text     | 1.14 | 1.95  | 2.44  |
| compress 100 KB text    | 3.12 | 12.21 | 10.85 |
| compress 500 KB text    | —    | —     | 21.54 |
| compress 1 MB text      | 2.61 | —     | —     |
| decompress 10 KB text   | 6.98 | 7.32  | 7.32  |
| decompress 100 KB text  | 5.78 | 5.74  | 5.43  |
| decompress 1 MB text    | 7.98 | —     | —     |
| compress 50 KB binary   | —    | 10.46 | 10.46 |
| decompress 50 KB binary | —    | 10.46 | 14.65 |

(`_bz2`/`_lzma`/`_zstd` have no Brython equivalent, so the columns measure
absolute throughput, not speedup. The writable-bytes bridge refactor
roughly doubled decompress throughput vs the prior baseline by skipping
one O(n) copy on the output path.)

**`array`** (5000-element typed array, ms/op):

| Operation                      | Wastdlib | Brython list (best alt) | Verdict   |
| ------------------------------ | ------- | ----------------------- | --------- |
| tobytes 5000 ints              | 7.5     | 51.4 (manual pack)      | **6.9×**  |
| tobytes + frombytes round-trip | 6.7     | 85.1                    | **12.7×** |
| construct array('i', 5000)     | 6.2     | ~0 (list copy)          | list wins |
| extend 10000 ints              | 28.0    | 0.5                     | list wins |

Brython's own `Lib/array.py` is broken (rejects valid typecodes, missing
methods). Wastdlib's `array` is the first working typed array in Brython.

**`_json`** (`encode_basestring`, scaling with input size):

| Input size            | Wastdlib ms | Brython pure-Py ms | Speedup    |
| --------------------- | ---------- | ------------------ | ---------- |
| 11 chars ASCII        | 0.014      | 0.358              | **25.57×** |
| 1300 chars ASCII      | 0.110      | 22.88              | **208×**   |
| 1700 chars w/ escapes | 0.090      | 24.12              | **268×**   |
| 11000 chars Lorem     | 0.450      | 149.45             | **332×**   |

Largest single-operation speedup measured on any ported module. The C
state machine asymptotes to ~24 MB/s of JSON encoding while pure-Python
char-by-char saturates around 75 KB/s — interpreter overhead vs native
string scan.

**Brython has a broken `_json` module**: a user (the author of this project)
filed a Brython issue 3 months before this port noting that
`json.loads(invalid_xml_string)` raised `json.decoder.JSONError` (an
exception that doesn't exist in CPython) instead of `JSONDecodeError`.
Brython's `_json` is also API-incompatible with CPython — Brython's
`_json.loads(s, **kw)` exists but CPython's `_json` has no top-level
`loads`. Wastdlib's `_json` ships the genuine CPython API. Combined with
bundling `Lib/json/`, `import json` would resolve that filed Brython
issue automatically.

**`math`** (int-heavy functions vs pure-Python equivalents):

| Operation          | Wastdlib ms | Brython ms | Speedup      |
| ------------------ | ---------- | ---------- | ------------ |
| factorial(200)     | 0.024      | 0.289      | **12×**      |
| factorial(500)     | 0.066      | 0.773      | **11.7×**    |
| gcd(10**18, 7**18) | 0.008      | 0.156      | **19.4×**    |
| isqrt(10**18)      | 0.004      | 0.089      | **22.2×**    |
| isqrt(10**40)      | 0.020      | 0.178      | 8.9×         |
| prod(range(1,100)) | 0.289      | 0.126      | 0.44× (loss) |

The int-heavy helpers (factorial, gcd, isqrt) win 9-22× because CPython's
C implementation does the BigInt arithmetic in tight C loops — exactly the
pattern Wastdlib thrives on. The trig/log floats (sin/cos/sqrt) are
delegated to libm and would barely beat Brython (browser `Math.sin` is
the same hardware op). `prod` loses because iteration crosses the bridge
per element — canonical case of the work-density anti-pattern (each
multiplication walks JS → bridge → C → bridge → JS).

**`cmath`** (complex math) — smoke 8/8. Historically the bench was noisy
across runs because each iteration used to leak a Brython complex wrapper
into the bridge's handle map (the sentinel handle-map leak — since fixed
by handle scopes, see *What's next*). Across reruns the directional pattern
is consistent: complex→complex ops (sqrt/exp/log/sin/cos) tend to win
**2-3.5×** while scalar-returning ops (`phase`, `polar`) lose. Exact
speedup numbers await a bench rerun now that the leak is gone (the
historical noise floor was ~50% on cmath bench results). Brython's bundled `cmath.py` is
broken at import time — Wastdlib's port is the first working complex math
for Brython users regardless.

**`_statistics`** (`_normal_dist_inv_cdf`, Wichura AS241 — ms/op):

| Quantile / branch                     | Wastdlib | Pure-Py AS241 | Speedup  |
| ------------------------------------- | ------- | ------------- | -------- |
| inv_cdf(0.5,   0, 1)   [central, q=0] | 0.00750 | 0.01800       | 2.4×     |
| inv_cdf(0.7,   0, 1)   [central]      | 0.00300 | 0.00900       | 3.0×     |
| inv_cdf(0.975, 0, 1)   [near-tail]    | 0.00500 | 0.01750       | 3.5×     |
| inv_cdf(0.999, 0, 1)   [near-tail]    | 0.00300 | 0.01900       | **6.3×** |
| inv_cdf(1−1e-15, 0, 1) [extreme tail] | 0.00350 | 0.02150       | **6.1×** |
| inv_cdf(0.95, 100, 15) [scaled]       | 0.00400 | 0.01500       | 3.8×     |

Textbook work-density curve: the tail branches evaluate one extra polynomial,
so more flops sit behind the single bridge crossing → bigger win. 1 call =
1 crossing for ~30-50 flops, so always a win, never close to break-even.

**`pyexpat`** (parse 5000 XML items, ~22 KB doc):

| Callback pattern                 | Time   | Throughput                    |
| -------------------------------- | ------ | ----------------------------- |
| Parse only (no Python callbacks) | 4.4 ms | **63 MB/s** (native libexpat) |
| Parse + 1 callback per element   | 364 ms | 767 KB/s (80× slower)         |
| Parse + 3 callbacks per element  | 678 ms | 412 KB/s (150× slower)        |

Cleanest demo of the work-density rule we have: same module, three patterns,
three orders of magnitude difference. The "parse only" path runs at native
libexpat speed because *no bridge crossings happen during the parse* — C
state machine just consumes bytes. Add a Python handler per element and
throughput collapses. Strategic implication: for XML in practice, the next
port should be `_elementtree` (the C accelerator for `xml.etree.ElementTree`)
which builds the DOM tree *inside C* from libexpat callbacks, so only one
bridge crossing happens at the end.

**Brython has no working XML parser**: own `pyexpat` errors on basic
`ParserCreate()` calls; `xml.etree`, `xml.dom`, `xml.sax` are not shipped.
Brython upstream explicitly labelled the XML gap **"won't fix"** — the
reasoning was that `pyexpat` is CPython C and no pure-Python port exists,
proposing browser `DOMParser` as the workaround. Wastdlib's `pyexpat` is the
first working XML parser for Brython users; combined with bundling CPython's
pure-Python `xml/` stdlib it lights up `xml.dom.minidom.parseString(...)`
and `xml.etree.ElementTree.fromstring(...)` end-to-end (etree at pure-Python
speeds until `_elementtree` is ported).

### What the bench results teach — the "work-density rule"

A clear pattern emerged across the ports. WASM-via-bridge wins when the C
code does substantial work **between** bridge crossings; it loses when the
operation is small and the bridge crossing dominates:

| Profile                                            | Speedup expected  | Examples                             |
| -------------------------------------------------- | ----------------- | ------------------------------------ |
| Compute-dense, custom C types, work in WASM        | **5-50×**         | `_decimal`, `_csv`, hashlib, zlib    |
| Single bulk call with O(N) inner loop in C         | **5-15×**         | `array.tobytes`, compress/decompress |
| Per-element bridge crossings (loop in JS, op in C) | **0.5-1×** (LOSS) | `math.prod`, `cmath.phase`           |

This is the second selection rule for choosing modules to port. The first
is **"engine separable from Py-API"** (the C side does substantive work
that doesn't constantly call back into Python). There used to be a third —
**"no static PyTypeObject"**: modules that define their types with 50-field
static struct initializers were incompatible with the bridge, because
wasthon.h reorders `struct _typeobject` and a positional initializer fills
the wrong slots. That rule fell with the bridge's `src/dtconvert.py` (see "How it's
built" below): `_datetime` — 7 static types, 227 positional slots — now
compiles verbatim and ships in both bundles (`datetime.date + timedelta`
drops from 146 µs interpreted to 1.7 µs, **86×**). The same conversion,
generalized, is what put real PyTorch on the bridge
([BryTorch](https://github.com/fgallaire/brytorch): 31 static types,
1029 slots).


## How it's built

The [Wasthon](https://github.com/fgallaire/wasthon) bridge is the leverage. It implements just enough of the public CPython
C-API to compile unmodified stdlib modules. From a `<x>module.c` in
CPython's `Modules/`, one `emcc` invocation produces an Emscripten ES6
module exposing `PyInit_<x>()`. A small Brython-side loader
(`loader/wasthon-loader.js`, from the bridge) instantiates it and registers it under
`__BRYTHON__.imported[<x>]` so `import <x>` from Python just works.

One module gets an extra build step. `_datetime` defines its 7 types as
**static `PyTypeObject` initializers** — positional C89 lists whose meaning
depends on the exact field order of `struct _typeobject`, which wasthon.h
reorders. Compiled raw, every slot lands in the wrong field. So
the bridge's `src/dtconvert.py` rewrites the 227 positional slots into **C99 designated
initializers** (`.tp_dealloc = ...`) at copy time, keyed on CPython's
canonical slot order. It is a deterministic source-to-source pass, the
output is ordinary diffable C, and the source stays byte-for-byte upstream
CPython in the repo — the same zero-fork rule as every other module. The
runtime half (`_PyDateTime_InitTypes`, singleton registration, the real
packed-struct `datetime.h` for capsule consumers like pandas) rides in
`datetime_exec` via `build.sh`.

```
                ┌──────────────────────────────────────────────┐
   <x>module.c  │  emcc  →  PyInit_<x>() exported via WASM     │
   (CPython,    │     ↑                                        │
   unmodified)  │  wasthon.h + wasthon.c + wasthon.js          │
                │  (CPython C-API replicated atop Brython)     │
                └──────────────────────────────────────────────┘
                                  ↑
                          Brython runtime
                          (__BRYTHON__, _b_)
                                  ↑
                          User Python:
                              import _decimal
                              d = _decimal.Decimal('3.14') * 2
```


## Running it

Prerequisites:

- [Emscripten SDK](https://emscripten.org/) is installed automatically into
  `./external/emsdk/` on first build (pinned to 5.0.7); only `curl` or `wget`
  needs to be on PATH up front
- `make` (used by `emmake make` to build liblzma and libzstd)
- Python 3 (for `python3 -m http.server`)
- Brython is loaded by the loader pages from the vendored build in
  `loader/brython/` (stock 3.14.1 + the patches in `BRYTHON_FIX.md`); pass
  `?brython=cdn` on any page to load stock `brython@3.14.1` from jsDelivr — no
  local checkout needed either way

Source trees for CPython and the C libraries (bzip2, expat, xz, zstd) are
**downloaded automatically** by `build.sh` if not already present. Defaults
land them in `./external/<libname>` (gitignored). Override via env vars to
point at an existing checkout outside the repo:

```
CPYTHON_SRC, EXPAT_DIR, ZSTD_DIR, XZ_DIR, BZIP2_DIR
```

Build any module via the wrapper script (after activating emsdk):

```bash
cd wastdlib
./build.sh _sha2          # any of the 25 known modules → build/_sha2.{mjs,wasm}
./build.sh _sha2 _decimal # several specific modules in one go
./build.sh all            # everything as per-module .mjs/.wasm
                          # (~45 s once libs are cached; first run downloads + builds the libs too)
./build.sh wastdlib        # light bundle: 23 modules in build/wastdlib.{mjs,wasm} (~1 MB)
                          # — drops the three specialists (unicodedata, _zstd, _sqlite3)
./build.sh wastdlib-full   # full bundle: 26 modules in build/wastdlib-full.{mjs,wasm} (~3 MB)
```

The per-module target is best for dev, bench, and incremental work — each
module is fetched only if imported, and rebuilds are cheap. The bundled
targets are the "drop one script tag into your HTML" deliverable: one
fetch, one WASM instance, shared bridge runtime. `wastdlib` is the default
(~1 MB / 348 KB gzip); `wastdlib-full` adds the three specialists
`unicodedata` (full Unicode DB), `_zstd` (libzstd), and `_sqlite3`
(SQLite 3.46.1 + FTS5/RTREE/JSON1) — together responsible for most of the
full bundle's extra weight. Users who need any of them can also load the
per-module .wasm add-on alongside `wastdlib`. See `loader/test-wastdlib.html`
and `loader/test-wastdlib-full.html` for working bundle pages.

**Compile flags.** Modules are compiled with `emcc -O3` by default. The
project's positioning is perf-first — the C accelerators only earn their
wasm footprint if they beat Brython's pure-Python implementations by
large margins. `_sqlite3` is the single documented exception: built with
`-Oz` (halves the wasm — 1.35 MB → 730 KB — with negligible runtime cost
in practice, since SQLite is bridge-bound and carries large cold-path
features like FTS5/RTREE/JSON1 that aren't on the query hot loop). This
size cut is what made bundling `_sqlite3` in `wastdlib-full` viable.

**Link flags.** Links use `emcc -O2` with the standard runtime exports.
Three per-target deviations target the stack and the heap ceiling:

- **`-sSTACK_SIZE=4MB`** on `_decimal`, `_pickle`, `pyexpat` and both
  Wastdlib bundles. The sizing rule is **"match the legitimate use
  case, not just the tests"**: each of these modules has a stack-heavy
  code path that's part of its public contract — arbitrary-precision
  arithmetic on big numbers (`_decimal`), deeply-nested XML (`pyexpat`),
  deep object-graph serialization (`_pickle`). The test suite plateaus
  are tighter (e.g. `pyexpat` clears all its tests at 1 MB), but real
  workloads can push further — DOM trees with hundreds of nesting
  levels, recursive object graphs, `c.prec` set very high — so we bump
  everyone to 4 MB for headroom on legitimate inputs. **Zero `.wasm`
  byte cost** — STACK_SIZE is a runtime reservation, not embedded
  bytes. ALLOW_MEMORY_GROWTH=1 means the reservation only takes
  effect on actual use; idle code paths pay nothing.
- **`-sSTACK_OVERFLOW_CHECK=2`** on `_decimal` standalone and on the
  `wastdlib-full` bundle (which contains `_decimal`). libmpdec is the
  single module that can still push the stack at extreme inputs even
  with 4 MB; the per-prologue guard turns any remaining overflow into
  a clean Python exception instead of silent memory corruption.
  Level 1 (end-of-program sentinel) is too late. Size cost:
  `_decimal.wasm` +4.3%, `wastdlib-full.wasm` +~2%. NOT applied to the
  light `wastdlib` bundle (which doesn't ship `_decimal`) nor to
  `_pickle`/`pyexpat` standalone — the 4 MB headroom alone clears
  everything we've measured for those, and the per-prologue guard
  carries a small runtime perf cost.
- **`-sMAXIMUM_MEMORY=4GB`** on `_lzma` standalone and both Wastdlib
  bundles. The default 2GB growth ceiling was the
  real wall behind most of test_lzma's out-of-memory failures: liblzma's
  preset-6 encoder allocates ~94 MB per instance, and the *cumulative*
  heap pressure of a long-running page (the bridge's known handle-map /
  sentinel retention, see the GC section) pushes the total toward the
  cap, where `ALLOW_MEMORY_GROWTH` can grow no further and `malloc`
  starts returning NULL. 4 GB is the wasm32 maximum; the `maximum` is a
  virtual-address-space reservation, not an allocation, so it costs
  nothing on 64-bit hosts (32-bit user agents may fail to instantiate —
  considered acceptable in 2026). This buys headroom, it does not fix
  the retention. Re-evaluated once the per-call arena work landed
  (handle scopes, 2026-06-12): with retention gone, a 2GB build still
  measures −18 on test_lzma and times out test_pickle — the ceiling is
  real allocation appetite (liblzma's larger presets, pickle's biggest
  payloads), not bridge retention. 4GB is the permanent setting; the
  price is the unsigned-pointer discipline required of hand-written JS
  above the 2GB address boundary.

The script handles all the per-module quirks: downloading missing source
trees, compiling external libraries (libexpat, liblzma, libzstd, bzip2,
libmpdec, HACL\*), and the emcc `EXPORTED_FUNCTIONS`/`EXPORT_NAME` for each
target module.

Serve and test:

```bash
# from the wastdlib directory
python3 -m http.server 8765
# open http://localhost:8765/loader/index.html
# → individual test-*.html and bench-*.html pages for each module
# → test-all.html for a one-click sequential sweep of every test page
```

Headless runs and CI:

```bash
pip install playwright
playwright install chromium
python3 test.py   # spawns http.server, drives test-all.html via Chromium,
                  # exits 0 on green / 1 on red
```

A GitHub Actions workflow (`.github/workflows/test.yml`) runs this on every
push to `main` and every pull request — it builds all modules, both bundles,
and runs the headless sweep. The first run is the slow one (downloads
CPython + SQLite + emsdk + all libs, cold-compiles everything); subsequent
runs benefit from the `external/` and `build/*.o` caches.


## What's next

Recent ports:

- [x] `binascii` — CPython C accelerator for hex / base64 / CRC encoding
      (`hexlify`/`unhexlify`, `b2a_base64`/`a2b_base64`, `crc32`, `crc_hqx`,
      `b2a_hex` with the 3.14 `sep`/`bytes_per_sep` API). 24 KB wasm, the
      smallest module ported so far. Bundled in `wastdlib` light by
      default (+13 KB marginal — encoding is a fundamental Python
      primitive used wherever bytes touch the network or the file
      system). Brython's `binascii` is pure-Python; Wastdlib replaces it
      transparently with bit-exact CPython behaviour. **Honest
      benchmark** (2 MB payloads, median of 10 runs): decode operations
      are **~4× faster** (`unhexlify` 3.9×, `a2b_base64` 4.3×) and
      `crc32` works at all — Brython's pure-Python `crc32` raises
      `TypeError: ord() expected a character, but int was found`, a
      latent bug in its impl that Wastdlib resolves in passing. Encode
      operations are mixed: `b2a_base64` ~1.8× faster, but
      **`hexlify` is actually slower than Brython** (~0.55×) because
      the bridge bandwidth cost on `bytes → hex string` (6 MB of
      transfer for 2 MB input) outweighs the C win on what is otherwise
      a trivial byte-to-char operation. Reporting that regression
      honestly rather than papering it over with a JS-side shortcut:
      Wastdlib's contract is "real CPython C behaviour", optimization
      of `hexlify`-as-pure-Python belongs upstream in Brython.
      Port surfaced two CPython-internal symbols not previously
      needed by the bridge: `_PyLong_DigitValue[256]` (char→digit
      lookup used by hex parsing, added verbatim from
      `Objects/longobject.c`) and `_Py_strhex_bytes_with_sep`
      (hex-with-separator formatter; the naming is misleading — the
      `bytes_` infix denotes the *return* type, not the input, so it
      returns a `bytes` object). Also hit a new instance of the Brython
      kwarg-falsy-default quirk: a direct top-level
      `b2a_base64(b'…', newline=False)` compiles to
      `b2a_base64(b'…', 0=False)` (the kwarg name '`0`' is the value of
      `False`). Previously thought lambda-only; now confirmed to fire
      from plain calls too.
- [x] `_sqlite3` — SQLite 3.46.1 amalgamation + CPython's full `_sqlite`
      hierarchy (Connection / Cursor / Row / Blob / PrepareProtocol /
      Statement / microprotocols / util). FTS5, RTREE, and JSON1 enabled
      for real-world usefulness; `:memory:` only for now (no persistence
      wired yet). Bundled in `wastdlib-full`, also loadable standalone for
      sites that only need a database. **Compiled with `emcc -Oz`** —
      the project's overall rule is `-O3` (perf-first, that's Wastdlib's
      value prop), but SQLite is a documented exception: -Oz halves the
      wasm (1.35 MB → 730 KB) with negligible cost in practice (SQLite
      is bridge-bound + carries large cold-path features like FTS5/RTREE/
      JSON1 that aren't on query hot loops). The size cut is what made
      bundling sqlite in `wastdlib-full` viable. Bench validation is still
      blocked on handle reclamation: `tp_dealloc` dispatch has since landed,
      but a loop-bench `con = connect()` drops the Connection at Python
      scope exit, outside the explicit `close()`/`with` contract, so its
      native context leaks (the documented scope-exit residual — not an
      auto-finalizer, which a synchronous run can't fire promptly anyway).
      A proper compare-loop bench waits on that scope-exit reclamation. Bridge growth from this port — chief
      among them: `forwardError` helper preserves the original Brython
      exception class through C boundaries (replaces ~30 sites of
      `setError(RuntimeError, e.message)` flattening); `Py_tp_call`
      slot wiring lets callable types dispatch through C (sqlite's
      `statement_cache(lru_cache(n))` flow needs this); `__wasthon_type__`
      on `tp_new`'d instances unlocks `PyObject_TypeCheck` for clinic
      `__init__` guards; new bridge symbols `_PyUnicode_AsUTF8NoNUL`,
      `PyUnicode_FSConverter`, `PyLong_AsUInt32`, `PySequence_Check`,
      `_PyErr_FormatFromCause`, `PyErr_Print`, `PyExc_Warning`, plus
      single-threaded GIL stubs and `PyArg_ParseTuple` format-char `'U'`.
- [x] `_pickle` — C accelerator for `pickle`. Round-trips ints (incl. BigInts),
      floats, str (incl. non-ASCII unicode), bytes, bool, None, list, tuple,
      dict, set, frozenset, and arbitrary nestings of these across all four
      protocols (0–5; default 5). Bit-exact with CPython 3.14. Bundled in
      `wastdlib` light (+57 KB marginal) since serialization is a fundamental
      Python primitive. Brython's own pickle is pure-Python (slow) and
      exposes a different API surface; Wastdlib's port is the genuine
      CPython API. Surfaced and fixed a stack of latent bridge bugs
      benefiting all future ports — chief among them: `PyObject_GetAttr`
      must fall back through `cls.tp_funcs` (Brython's `$getattr` only
      consults the class dict); `PyErr_NewException` must rebuild the MRO
      and inherit `tp_new`/`tp_init` (raise-time "$is_slot of undefined"
      otherwise); the bytes "C wrote into `__wasthon_cstr__` but `.source`
      still zero" sync must descend recursively into container return
      values; `PyTuple_New` must go through `tuple.$factory` (a tagged JS
      Array doesn't fully mimic a Brython tuple); `PyOS_snprintf` needed
      a real varargs implementation (it was a stub that copied the format
      string verbatim, so `%zd\n` from protocol-0 INT serialization went
      straight into the pickle stream). And the gem: `bind_builtin_type`
      was last-write-wins on the Brython-class key, so `PyODict_Type` was
      overwriting `PyDict_Type` — making `Py_TYPE(dict) == &PyDict_Type`
      silently false and pickle fall through to the reduce path. Latent
      for the previous 22 modules because none compared
      `Py_TYPE(obj) == &PyXxx_Type` directly.
- [x] `_decimal` (libmpdec) — arbitrary-precision math, 18-46× speedup.
- [x] `_csv` — full state machine port, 8-25× speedup.
- [x] Full compression family: `_bz2`, `_lzma`, `_zstd` alongside `_zlib`.
- [x] `array` — foundation for typed arrays / future numerics.
- [x] `pyexpat` (libexpat) — XML parser. Brython upstream had labelled the
      XML gap "won't fix" (no pure-Python pyexpat). Wastdlib's port resolves
      it: real CPython pyexpat, 63 MB/s on the parse-only path.
- [x] `_json` — JSON encoder/decoder C accelerator. **25-332× speedup**
      on encode_basestring vs pure-Python (largest single-op speedup of
      any port). Brython has its own incompatible `_json` with a known bug
      filed 3 months prior to this port — Wastdlib's port ships the real
      CPython API and resolves it.
- [x] `math` — int-heavy fns (factorial / gcd / isqrt / prod / comb / perm).
      **9-22× speedup** on the int helpers (factorial(500): 11.7×,
      isqrt(10**18): 22.2×). Float ops (sin/cos/sqrt) delegate to libm
      and gain nothing over browser `Math.X`. Exposed two latent generic
      bridge bugs: PyNumber_Multiply/Add silently produced JS Infinity on
      Number×Number overflow (now promotes to BigInt automatically), and
      `_PyLong_Lshift`/`_Rshift` were declared `size_t` in our header but
      CPython 3.14 uses `int64_t` — emcc ABI mismatch produced garbage
      shift values for the new BigInt-aware code paths.
- [x] `cmath` — complex math. 8/8 smoke. Bench numbers pending a rerun:
      the old 50%+ variance came from handle-map pressure — the sentinel
      handle-map leak behind that noise is now fixed (handle scopes — see [Wasthon](https://github.com/fgallaire/wasthon)'s
      *What's next*); directionally: complex→complex ops win ~2-3.5×,
      scalar-returning ops lose. Exposed the emcc wasm32
      ABI pattern for passing/returning
      small structs (Py_complex) by value via sret. Brython's own
      `cmath.py` is broken at import time — Wastdlib's is the first
      working complex math for Brython users regardless.
- [x] `_statistics` — `_normal_dist_inv_cdf` (Wichura AS241). 7/7 smoke.
      6/6 bench wins: 2.4× central, up to **6.3×** on the tail branches.
      Smallest port to date (15 KB wasm), single METH_FASTCALL function.
      Textbook validation of the work-density rule — one bridge crossing,
      ~30-50 flops inside C, never close to break-even.
- [x] Top-level `build.sh <module>|all|wastdlib|wastdlib-full|list` script. Bootstraps emsdk into
      `external/emsdk/` if missing, downloads CPython + libexpat + libxz +
      libzstd + bzip2 sources via `curl`/`wget` on first run, compiles all
      the per-module quirks (which CPython sources, which external library
      objects, include paths, `EXPORTED_FUNCTIONS`, `EXPORT_NAME`). On a
      fresh checkout: `./build.sh all` goes from zero to 23 `.wasm` in
      a single command (~45 s once libs are cached).

The "stdlib integration" layer — where the next wins live:

Wastdlib ports the C modules. The natural complement is wiring them to the
pure-Python stdlib wrappers that Brython users actually import. Each gap
below has a C module already in place; what's left is integration glue.

- [ ] **Bundle CPython's `Lib/xml/` into Brython.** `xml.dom.minidom`,
      `xml.etree.ElementTree`, `xml.sax`, `xml.parsers.expat` are all
      pure-Python — they just need `pyexpat` to exist, which it now does.
      `xml.dom.minidom.parseString(...)` and similar would light up
      immediately. Direct response to Brython's "won't fix" on XML.
- [ ] **Bundle CPython's `Lib/json/` into Brython.** `json/__init__.py`,
      `decoder.py`, `encoder.py`, `scanner.py` are pure-Python wrappers
      around `_json`. Brython has its own incompatible `json/` package
      with an open bug. Replacing it with CPython's gives the proper
      `JSONDecodeError`-with-context behaviour AND picks up Wastdlib's
      25-332× speedup automatically.
- [ ] **`_elementtree` (the C accelerator).** With it, `xml.etree`
      operates at near-libexpat speed (50-60 MB/s effective) because the
      DOM tree is built in C from libexpat callbacks — one bridge crossing
      at the end of the parse instead of one per element.
- [x] **Wire `_sre` into Brython's `re.py`.** Brython ships its own
      pure-Python regex code in `re.py`. Patch (or replace) `re.py` so it
      uses our `_sre` C module when available. More invasive than the XML
      case because Brython has an incumbent, not a gap.
      *Done by Pierre 2026-05-26: Brython's `Lib/re/` is now CPython
      3.14's `re/` package verbatim, importing `_sre` (auto-loaded via
      `$B.wasthonLoad` at boot). Confirmed all `tests/test_wasthon`
      pass.*

Module candidates worth porting are largely exhausted at this point. What
remains in CPython's stdlib falls into one of three buckets:

- **Structurally impossible in browser** — `_socket`, `_ssl`, `_thread`,
  `_ctypes`, `_curses`, `mmap`, `select`, `_asyncio`, heavy `_io`,
  `_multiprocessing` (no OS). (`_datetime` used to sit here under the
  static-`PyTypeObject` ban; `dtconvert.py` lifted the ban and it is now
  module #26.)
- **Fails the work-density rule** — `_functools` (`lru_cache`/`reduce`),
  `_operator`, `_queue` (no threads anyway). `_heapq` and `_bisect` were
  ported and dropped after benchmarks measured zero-or-negative gain.
- **Needs integration layer or non-trivial bridge work** — `itertools`
  (attempted, rolled back — exposed 4 transversal bridge gaps in series),
  `_elementtree` (needs `Lib/xml/` bundled), `_tokenize` (needs Parser/
  sources bundled), `_multibytecodec` + CJK codecs (needs codec system
  integration in Brython).

Future work is therefore in **depth** (better infra, integration) rather
than **breadth** (more modules).

## Acknowledgements

The crypto work rides on [HACL\*](https://project-everest.github.io/) —
formally verified C implementations bundled in CPython 3.13+. The zlib
build uses Emscripten's `madler/zlib` port. The compression trilogy bundles
[bzip2](https://sourceware.org/bzip2/) (Julian Seward),
[xz-utils](https://tukaani.org/xz/) (Lasse Collin and Igor Pavlov's LZMA),
and [zstd](https://facebook.github.io/zstd/) (Yann Collet, Meta). `_decimal`
embeds [libmpdec](https://www.bytereef.org/mpdecimal/) (Stefan Krah).
`pyexpat` rides on [libexpat](https://libexpat.github.io/) (James Clark
and successors).
`_sqlite3` bundles the [SQLite](https://www.sqlite.org/) amalgamation
(D. Richard Hipp; placed in the public domain).
Wasthon is mostly the plumbing that lets these libraries talk to Python
code translated to JavaScript by Brython, through a synthetic CPython
C-API implemented over the JavaScript runtime.

## License

Copyright (C) 2026 Florent Gallaire <fgallaire@gmail.com>

BSD 3-Clause License — same as Brython. See `LICENSE` for the full text
and `THIRD_PARTY.md` for the upstream components and their licenses.
