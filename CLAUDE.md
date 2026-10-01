# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

libmie is the MokyaInput Engine (MIE): a Bopomofo half-keyboard + English
prediction input method engine in portable C++11. It was extracted with full
history from `firmware/mie/` of
[tengigabytes/MokyaLora](https://github.com/tengigabytes/MokyaLora), where it
runs on the RP2350B's Core 1. MokyaLora still keeps its own copy; libmie is
the mainline for new engine work and is consumed by other front ends (e.g.
`tengigabytes/mokya-ime-android` as a git submodule).

## License — MIT

- Everything in this repository is MIT (see `LICENSE`). Put
  `SPDX-License-Identifier: MIT` at the top of every new source file.
- Never add code from, or a dependency on, Meshtastic or any GPL / LGPL
  source. MokyaLora links this library into an Apache-2.0 image and keeps a
  strict licence boundary.
- Dictionary / font *data* (libchewing-data `tsi.csv`, FrequencyWords,
  google-10000-english, GNU Unifont, MoE CSV) are third-party with their own
  licences. `tools/fetch_data.py` downloads them into `data_sources/`; never
  commit them or generated `.bin` files (both are gitignored).

## Compatibility rules (must keep libmie usable on MokyaLora hardware)

Changes are **additive only**:

1. Do not change existing `ImeLogic` public function signatures (add
   overloads instead).
2. Do not change `include/mie/keycode.h` values.
3. Do not change the MIE4 v4 dictionary format
   (`include/mie/composition_searcher.h`, `tools/gen_dict.py`).
4. Do not change the `LRU1` persistence format (`include/mie/lru_cache.h`).
5. RAM-sized constants keep their defaults (`ImeLogic::kMaxCandidates = 100`
   via `MIE_MAX_CANDIDATES`, `LruCache::kCap = 128` via `MIE_LRU_CAP`). Hosts
   override them only through the CMake cache variables, which apply PUBLIC
   compile definitions; a per-file `#define` silently breaks the ODR because
   both change `sizeof(ImeLogic)`.
6. The library builds as C++11 (`MIE_CXX_STANDARD` default) with no OS, SDK,
   exception or RTTI dependency, and `ImeLogic` does not allocate. Do not
   add platform headers to `include/` or `src/`.

Also keep in mind (from the RP2350 build): large scratch buffers in
`src/ime_search.cpp` are function-local `static`s tagged `MIE_PSRAM_BSS`,
and `MOKYA_MIE_PERF_TRACE` enables optional trace hooks. Both compile to
nothing on host builds; keep them working.

## Sources of truth

- Key layout: `kKeyTable` in `src/ime_keys.cpp` (dict byte = slot + 0x21;
  must match `_BPMF_KEYMAP_RAW` in `tools/gen_dict.py`).
- Keycodes: `include/mie/keycode.h`.
- Engine contract (threading, key routing, listener): comments in
  `include/mie/ime_logic.h`.
- v4 binary layout: header comment in `include/mie/composition_searcher.h`.
- Modes: exactly three — SmartZh, SmartEn, Direct.

## Build and test

```sh
# Configure + build + test (Linux / macOS)
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build --parallel
ctest --test-dir build --output-on-failure

# If FetchContent cannot download GoogleTest (proxy):
git clone --depth 1 --branch v1.14.0 https://github.com/google/googletest /tmp/googletest
cmake -S . -B build -DFETCHCONTENT_SOURCE_DIR_GOOGLETEST=/tmp/googletest

# Other standards / overrides exercised by CI
cmake -S . -B build17 -DMIE_CXX_STANDARD=17
cmake -S . -B build-ovr -DMIE_MAX_CANDIDATES=200 -DMIE_LRU_CAP=512

# Python tool tests
python -m pytest tests/test_gen_dict.py tools/test_pack_dict_blob.py

# Interactive REPL
./build/mie_repl
```

`ImeV4Dispatch.RealDict_SmokeTest` runs only when a generated
`dict_mie_v4.bin` exists (`data/dict_mie_v4.bin` or `$MIE_TEST_DICT_V4`).

Generate the dictionary:

```sh
python tools/fetch_data.py --data-dir data_sources
python tools/gen_dict.py --libchewing data_sources/tsi.csv --zh-max-abbr-syls 4 \
    --en-wordlist data_sources/en_50k.txt \
    --v4-output data/dict_mie_v4.bin --output-dir /tmp/mie_v2_throwaway
```

CI (`.github/workflows/ci.yml`) builds and tests gcc/clang × C++11/C++17,
one leg with size overrides, and the Python tool tests. Every change must
keep all legs green; add tests for new behaviour.

## Conventions

- Code comments and documentation are in English. Chinese appears only in
  test data and IME output examples.
- Match the surrounding style: 4-space indent, `snake_case` methods,
  trailing-underscore members, `k`-prefixed constants.
- Legacy pieces kept on purpose: the MIED v2 `TrieSearcher` path (also used
  for the embedded English dictionary), the MDBL v2 CMake data targets, and
  the v1 C API (`include/mie/mie.h`, `src/mie_c_api.cpp`, excluded from the
  build). Do not delete them without a migration plan for MokyaLora.
