# libmie — MokyaInput Engine

Portable input method engine for Traditional Chinese (Bopomofo half-keyboard)
and English (two-letters-per-key prediction), written in C++11 with no OS or
SDK dependency.

libmie was extracted (with full commit history) from `firmware/mie/` of
[MokyaLora](https://github.com/tengigabytes/MokyaLora), an open-hardware
Meshtastic feature phone built on the RP2350B, where it runs on Core 1
with a 36-key physical keypad. It is kept platform-neutral so the same
engine can also back other front ends (e.g. an Android input method).
MokyaLora currently keeps its own copy of the engine; libmie follows the
compatibility rules below so changes can flow back to the hardware.

---

## Features

- **Three input modes**, cycled by `MOKYA_KEY_MODE`
  (SmartZh → SmartEn → Direct → SmartZh):
  - **SmartZh (中)** — Bopomofo half-keyboard. 20 keys each carry two (one
    key: three) Bopomofo symbols; a short tap matches any of them. A long
    press (the event producer sets `MOKYA_KEY_FLAG_LONG_PRESS`) pins the
    first symbol, and further long presses of the same key within 800 ms
    cycle to the next one. Producers that know the exact phoneme (e.g. a
    full Zhuyin keyboard, where every phoneme has its own key) set
    `MOKYA_KEY_FLAG_PHONEME(idx)` instead.
    Abbreviated input (initials only) expands to whole words, SPACE marks
    tone 1, and partial commits keep the unmatched tail.
  - **SmartEn (EN)** — dictionary prediction over the two letters printed
    on each key, with sentence-aware auto-capitalisation and spacing. Row-0
    keys multi-tap digits.
  - **Direct (ABC)** — plain multi-tap (`a → s → A → S`, digits `1 → 2`)
    for passwords and out-of-dictionary text.
- **Punctuation** — SYM1 short press inserts `，` / `,`; a long press opens a
  4 × 4 symbol picker (`「」『』（）【】，。、；：？！…`). SYM2 multi-taps
  `。？！` / `.?!`.
- **MIE4 v4 dictionary** — a single binary blob (magic `MIE4`). Chinese
  words are stored as character + reading references and composed at search
  time; the English dictionary is embedded in the same blob. Loadable from a
  file or zero-copy from memory (flash / PSRAM / an mmapped asset).
- **Personalised LRU** — recently committed rare readings surface first.
  State serialises to a small `LRU1` blob for persistence.
- **Listener API** — the front end pushes `KeyEvent`s and periodic `tick()`s;
  the engine reports commits, deletes and cursor moves through
  `IImeListener`. The front end owns the text buffer; the engine owns only
  the pending composition and candidates. Time is injected by the caller
  (`KeyEvent::now_ms`), so the engine has no clock of its own.
- **Embedded-friendly** — fixed-size state (no heap use in `ImeLogic`), no
  exceptions or RTTI required, builds as a plain static library.

---

## Repository layout

```
libmie/
├── include/mie/
│   ├── ime_logic.h            — ImeLogic, IImeListener, PendingView (main API)
│   ├── composition_searcher.h — MIE4 v4 dictionary search (+ format spec)
│   ├── trie_searcher.h        — MIED v2 search (used for the embedded English dict)
│   ├── lru_cache.h            — personalised LRU + LRU1 persistence format
│   ├── keycode.h              — canonical keycodes (C-compatible)
│   ├── hal_port.h             — KeyEvent
│   ├── utf8.h                 — UTF-8 helpers (C-compatible)
│   └── mie.h                  — legacy v1 C API (not built, see below)
├── src/
│   ├── ime_logic.cpp          — constructors, process_key dispatcher, tick, DPAD/DEL
│   ├── ime_keys.cpp           — kKeyTable (key layout) + syllable position counter
│   ├── ime_smart.cpp          — SmartZh / SmartEn handler
│   ├── ime_direct.cpp         — Direct mode, multi-tap, SYM1 / SYM2
│   ├── ime_search.cpp         — v4 / v2 search dispatch, LRU merge
│   ├── ime_display.cpp        — pending composition string
│   ├── ime_commit.cpp         — commit / partial commit
│   ├── composition_searcher.cpp, trie_searcher.cpp, lru_cache.cpp, utf8.c
│   └── mie_c_api.cpp          — legacy v1 C API (excluded from the build)
├── hal/pc/                    — terminal key reader for mie_repl
├── tools/
│   ├── fetch_data.py          — download dictionary / font sources
│   ├── gen_dict.py            — build the MIE4 v4 dictionary
│   ├── gen_font.py            — build the MIEF bitmap font (MokyaLora UI)
│   ├── pack_dict_blob.py      — legacy MDBL v2 packer
│   └── mie_repl.cpp           — interactive REPL on a PC terminal
├── tests/                     — GoogleTest suites + gen_dict pytest
└── data/                      — generated assets (not committed)
```

---

## Build

Requirements: CMake ≥ 3.20, a C++11 compiler (GCC, Clang, MSVC 2019+), and
Python ≥ 3.8 for the data tools only. The test build downloads GoogleTest
v1.14.0 through `FetchContent`.

```sh
# Linux / macOS
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --parallel
ctest --test-dir build --output-on-failure
./build/mie_repl                       # interactive REPL

# Windows (VS Build Tools 2019; project on a local drive)
cmake -S . -B build -G "Visual Studio 16 2019" -A x64
cmake --build build --config Debug --parallel
cmake --build build --config Debug --target RUN_TESTS
```

If `FetchContent` cannot download GoogleTest (e.g. behind a proxy), clone it
and point CMake at the checkout:

```sh
git clone --depth 1 --branch v1.14.0 https://github.com/google/googletest /path/to/googletest
cmake -S . -B build -DFETCHCONTENT_SOURCE_DIR_GOOGLETEST=/path/to/googletest
```

`ImeV4Dispatch.RealDict_SmokeTest` is skipped unless a generated
`dict_mie_v4.bin` is found (`data/dict_mie_v4.bin`, or the path in the
`MIE_TEST_DICT_V4` environment variable).

Python tool tests: `python -m pytest tests/test_gen_dict.py tools/test_pack_dict_blob.py`.

### CMake options

| Option | Default | Purpose |
|--------|---------|---------|
| `MIE_CXX_STANDARD` | `11` | C++ standard for the library, REPL and tests. |
| `MIE_MAX_CANDIDATES` | *(empty → 100)* | Overrides `ImeLogic::kMaxCandidates`. |
| `MIE_LRU_CAP` | *(empty → 128)* | Overrides `LruCache::kCap`. |
| `MIE_BUILD_HOST_TOOLS` | ON when top-level and not cross-compiling | Build `mie_repl`, the tests and the `mie_data_*` targets. |
| `MIE_DATA_SOURCES_DIR`, `MIE_UNIFONT_OTF`, `MIE_MOE_CSV` | *(empty)* | Inputs for the optional `mie_data_*` targets. |

`MIE_MAX_CANDIDATES` and `MIE_LRU_CAP` change `sizeof(ImeLogic)`. They are
applied as **PUBLIC** compile definitions on the `mie` target so every
translation unit that links it agrees on the layout. Do not define them
per file. The defaults are what MokyaLora's RP2350 build is budgeted for.

### Using libmie from another CMake project

```cmake
add_subdirectory(path/to/libmie)          # e.g. a git submodule
target_link_libraries(my_target PRIVATE mie)
```

As a subdirectory, or whenever `CMAKE_CROSSCOMPILING` is set (Android NDK,
Pico SDK), only the `mie` static library is defined; the REPL, tests and
data targets are skipped (`MIE_BUILD_HOST_TOOLS=OFF`), so the parent build
does not download GoogleTest.

---

## Dictionary

```sh
python tools/fetch_data.py --data-dir data_sources --only tsi.csv en_50k.txt
python tools/gen_dict.py \
    --libchewing data_sources/tsi.csv \
    --zh-max-abbr-syls 4 \
    --en-wordlist data_sources/en_50k.txt \
    --v4-output data/dict_mie_v4.bin \
    --output-dir /tmp/mie_v2_throwaway
```

`--v4-output` is the single file the engine needs. Without `--en-wordlist`
the blob has no English section and SmartEn offers no predictions (digit and
multi-tap input still work). The v2 files written to `--output-dir` are a
by-product of the build and can be discarded. The full blob is about 4 MB.

The `mie_data_*` CMake targets still produce the **retired** MDBL v2 pack
(`dict_dat.bin`, `dict_values.bin`, `en_dat.bin`, `en_values.bin`). They are
kept for bisecting only; see `data/README.md`.

The binary layout is documented at the top of
`include/mie/composition_searcher.h`.

Data sources are third-party and are **not** covered by libmie's MIT
licence. Check their licences before redistributing a generated dictionary:
libchewing-data `tsi.csv` (LGPL-2.1-or-later per its file header),
hermitdave/FrequencyWords, first20hours/google-10000-english, GNU Unifont,
and the optional MoE dictionary CSV.

---

## C++ usage

```cpp
#include <mie/composition_searcher.h>
#include <mie/ime_logic.h>

struct Listener : mie::IImeListener {
    void on_commit(const char* utf8) override       { /* insert at cursor */ }
    void on_delete_before() override                { /* delete 1 char before cursor */ }
    void on_cursor_move(mie::NavDir d) override     { /* move cursor */ }
    void on_composition_changed() override          { /* repaint pending + candidates */ }
};

mie::CompositionSearcher zh;
zh.load_from_memory(blob, blob_size);        // blob must outlive zh (no copy)

mie::TrieSearcher en;                        // English dict embedded in the v4 blob
const uint8_t *dat, *val; size_t dat_n, val_n;
bool has_en = zh.english_sections(&dat, &dat_n, &val, &val_n) &&
              en.load_from_memory(dat, dat_n, val, val_n);

mie::ImeLogic ime(zh, has_en ? &en : nullptr);   // v4-only constructor
Listener listener;
ime.set_listener(&listener);

mie::KeyEvent ev;
ev.keycode = MOKYA_KEY_C; ev.pressed = true; ev.now_ms = now(); ev.flags = 0;
ime.process_key(ev);                         // also send key-up for SYM1
ime.tick(now());                             // every ~20 ms, see rules below

mie::PendingView pv = ime.pending_view();    // composition string + style
for (int i = 0; i < ime.page_cand_count(); ++i) ime.page_cand(i).word;
```

Rules the front end must follow:

- Call `ImeLogic` from one thread only, and never from inside a listener
  callback (no re-entrancy).
- `now_ms` must be monotonic and share one time base with `tick()`.
- Call `tick()` about every 20 ms at least while `has_pending()` is true
  (multi-tap auto-commit) **or SYM1 is held down** (the long-press picker
  opens from `tick()`, and `has_pending()` stays false while SYM1 is held).
- Filter `MOKYA_KEY_BACK` before dispatch (the engine ignores it; it belongs
  to the UI).
- After any edit the engine did not make (cursor moved, text pasted or
  deleted externally), call `abort()` and `set_text_context()` with up to two
  code points before the cursor.
- Persist the LRU with `lru_serialized_size()` / `serialize_lru()` and
  restore it with `load_lru()`.
- Idle `OK` commits `"\n"`; idle `SPACE` commits `" "`.

The older v2 constructor `ImeLogic(TrieSearcher& zh, TrieSearcher* en)` plus
`attach_composition_searcher()` is still supported and behaves identically.

---

## Key layout

Keycodes are defined in `include/mie/keycode.h`; the input-key layout is
`kKeyTable` in `src/ime_keys.cpp` (the source of truth if this table and the
code ever disagree). On MokyaLora the keycodes map to a 6 × 6 matrix:

| Row | Col 0 | Col 1 | Col 2 | Col 3 | Col 4 | Col 5 |
|-----|-------|-------|-------|-------|-------|-------|
| 0 | `1` ㄅㄉ · 1 2 | `3` ˇˋ · 3 4 | `5` ㄓˊ · 5 6 | `7` ˙ㄚ · 7 8 | `9` ㄞㄢㄦ · 9 0 | FUNC |
| 1 | `Q` ㄆㄊ · q w | `E` ㄍㄐ · e r | `T` ㄔㄗ · t y | `U` ㄧㄛ · u i | `O` ㄟㄣ · o p | SET |
| 2 | `A` ㄇㄋ · a s | `D` ㄎㄑ · d f | `G` ㄕㄘ · g h | `J` ㄨㄜ · j k | `L` ㄠㄤ · l | BACK |
| 3 | `Z` ㄈㄌ · z x | `C` ㄏㄒ · c v | `B` ㄖㄙ · b n | `M` ㄩㄝ · m | `\` ㄡㄥ · — | DEL |
| 4 | MODE | TAB | SPACE | SYM1 `，` | SYM2 `。.？` | VOL+ |
| 5 | UP | DOWN | LEFT | RIGHT | OK | VOL− |

Each input key shows `MOKYA_KEY_x` name, Bopomofo symbols, then digits
(row 0, SmartEn/Direct) or letters (rows 1–3). Function keys:

| Key | Behaviour |
|-----|-----------|
| MODE | Commit pending input, then cycle SmartZh → SmartEn → Direct. |
| SPACE | Idle: insert a space. SmartZh while composing: tone-1 marker. |
| OK | Commit the selected candidate / pending multi-tap; idle: insert `"\n"`. |
| DEL | Delete the last pending key; with nothing pending: `on_delete_before()`. |
| DPAD | Move through candidates; with none showing: `on_cursor_move()`. |
| TAB | Next candidate page. |
| SYM1 | Short: `，`/`,`. Long (≥ 500 ms, detected by the engine): symbol picker. |
| SYM2 | Multi-tap `。？！` / `.?!`. |
| BACK | Never consumed by the engine; reserved for the UI. |

FUNC, SET, VOL±, and POWER are not used by the engine.

---

## Compatibility with MokyaLora

libmie must stay usable on MokyaLora's RP2350 Core 1. Changes are additive:

- Do not change existing `ImeLogic` public function signatures, `keycode.h`
  values, the MIE4 v4 dictionary format, or the `LRU1` persistence format.
- Keep RAM-sized constants (`kMaxCandidates = 100`, `LruCache::kCap = 128`)
  at their defaults; hosts override them through CMake, not by editing them.
- Keep the library buildable as C++11 with no OS, SDK, heap-heavy or
  exception-dependent code.

The legacy C API (`include/mie/mie.h`, `src/mie_c_api.cpp`) targets the
retired v1 engine and is excluded from the build until it is rewritten
against the listener API. Use the C++ API directly (e.g. from JNI).

---

## License

MIT — see [LICENSE](LICENSE). libmie has no dependency on Meshtastic or any
GPL-licensed code. Dictionary and font data generated by `tools/` come from
third-party sources with their own licences (see [Dictionary](#dictionary)).
