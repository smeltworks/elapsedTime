# Platform survey — gap list

Each canonical platform example, attempted in **both** C-Next and C/C++, built
against elapsedTime as a real `lib_deps` consumer. This file records what fought
back. **No fixes in this pass** — the point is a list to work through before
v1.0.0 is cut.

Toolchain note: the survey (2026-09-20) was built with c-next from `main`,
because the released 0.3.0 could not transpile the library.

**2026-10-25: migrated to C-Next 0.4.0.** Each example's timer and `ledOn`
globals are now `mut`, and `LED_PIN` lost its `const` (E0900, E0898). Every
`.cnx` example transpiles on the released `cnext` 0.4.0 and the generated C is
committed. To regenerate one:

- **Name the target.** 0.4.0 refuses a program with no target (E0515), and
  reading it from `platformio.ini` works only for boards its catalog names.
  `teensy40` maps by name and `uno` through its `atmelavr` platform.
  `esp32dev` and `nucleo_f401re` do not (E0510), so
  those four examples set `"target"` in `cnext.config.json` (`esp32` or
  `stm32f4`).
- **Generate the compile database** for STM32 HAL and both ESP32 examples:
  `pio run -t compiledb`. It is how c-next finds the framework's headers and
  `-D` flags (see G4).
- **ESP32 only: name the compiler.** The database names `xtensa-esp32-elf-gcc`
  without a path, and PlatformIO keeps its toolchain off `PATH`, so set
  `CNEXT_CROSS_COMPILER` to the compiler under
  `~/.platformio/packages/` (`toolchain-xtensa-esp32` for the Arduino core,
  `toolchain-xtensa-esp-elf` for ESP-IDF). Filed as
  [c-next#2356](https://github.com/smeltworks/cnx/issues/2356); after the
  rename (#2166) the variable is `CNX_CROSS_COMPILER`.
- **Delete the stray `src/ElapsedTime.h`** each run leaves in the library (G3).

The examples now take the library as `lib_deps = symlink://../../..`. With
`file://`, PlatformIO copied the whole repo into each example's
`.pio/libdeps`, ignoring `library.json`'s `export` list, and each copy held the
other examples' `.pio` directories, copies included. Rebuilding all eleven one
after another nested those copies to about 300 GB.

## Build matrix

| Platform | Framework | Canonical example | C-Next | C/C++ |
| --- | --- | --- | --- | --- |
| Arduino AVR (Uno) | arduino | `BlinkWithoutDelay` | **PASS** | **PASS** |
| ESP32 | arduino | `BlinkWithoutDelay` | **FAIL** — G1, G2 | **FAIL** — G1 |
| ESP32 | espidf | `esp_timer` periodic | **PASS** (G4 friction) | **PASS** |
| STM32 Nucleo F401RE | stm32cube | CubeMX blink on `HAL_GetTick` | **PASS** (G4 friction) | **PASS** |
| STM32 Nucleo F401RE | arduino | `BlinkWithoutDelay` | **PASS** | not attempted |
| Teensy 4.0 | arduino | `BlinkWithoutDelay` vs `elapsedMillis` | **PASS** | **PASS** |

All eleven builds re-run on 2026-10-25 against C-Next 0.4.0 give the same
verdicts. ESP32 Arduino still fails as G1 and G2 record: in C++ with the
`invalid conversion` error, and in C-Next with the `-Wincompatible-pointer-types`
warning followed by `undefined reference to 'setup()'`.

On AVR and Teensy, both variants are the same size to the byte — AVR 42 B RAM /
1882 B flash, Teensy 8420 B code (0.4.0, 2026-10-25). The survey measured AVR
30 B / 1072 B and Teensy 8116 B. The growth is #10's larger `Timer` (the tick's
range fields and a `u64` firing mark), and it lands equally in both languages.

Not attempted: Raspberry Pi Pico (SDK), Zephyr, bare FreeRTOS. Each would add
another instance of a gap class already characterised below (G4 for header
resolution, G5 for tick width) rather than a new one, so they were skipped
rather than padded in. Pico is still worth doing for real once G5 is settled,
because `get_absolute_time()` returns a struct and `time_us_64()` a native u64.

---

## Status — G1 waits on c-next v0.4.1 (updated 2026-10-25)

Investigating G1 traced its root cause upstream (2026-09-20). Re-checking G3 and
G4 against c-next 0.4.0 and `main` (2026-10-18) found three more. Six c-next
issues were filed from this survey, all milestoned **v0.4.1**.

**2026-10-25:** #1619, #1618 and #1620 are closed on c-next `main`
(2026-10-23/24) but unreleased; v0.4.1 has not shipped. On 0.4.0, `tick: millis`
still passes silently. Migrating to 0.4.0 added a seventh issue,
[c-next#2356](https://github.com/smeltworks/cnx/issues/2356) (G4: the compile
database's compiler is not found off `PATH`).

| Issue | Gap | What it covers |
| --- | --- | --- |
| [c-next#1619](https://github.com/smeltworks/cnx/issues/1619) | G1 | **G1's root cause.** E0879 is silent for a scope-declared callback type across an include — exactly this library's shape — and for any function from a C header. The issue did not say what verdict `tick: millis` should get, so [a comment](https://github.com/smeltworks/cnx/issues/1619#issuecomment-6249021830) pins it: C type identity under the target's own compiler, the same verdict gcc gives. That rejects it on ESP32 with xtensa gcc 8.4 and accepts it on AVR, ARM, and ESP32 with gcc 13 or later (see G1). |
| [c-next#1618](https://github.com/smeltworks/cnx/issues/1618) | G1 | ADR-024 conversions enforced per-position: 6 of 10 positions leak, and every position leaks from a C source. Found while checking whether the ESP-IDF `now[0, 32]` adapter was actually required — it is correct, but nothing would have forced it. |
| [c-next#1620](https://github.com/smeltworks/cnx/issues/1620) | G1 | Why neither surfaced: adr-matrix occupancy is per-ADR rather than per-diagnostic, so one rule greens a cell for every code an ADR declares. |
| [c-next#2151](https://github.com/smeltworks/cnx/issues/2151) | G3 | A consumer's run rewrites an included library's `.c` for the *consumer's* target, and writes a second header with a different include guard into the library's tree. |
| [c-next#2153](https://github.com/smeltworks/cnx/issues/2153) | G4 | An include cycle in ESP-IDF's FreeRTOS headers makes the `-imacros` retry import `task.h`'s own include guard, so `task.h` reads as empty and `vTaskDelay` is E0422. |
| [c-next#2154](https://github.com/smeltworks/cnx/issues/2154) | G4 | The `cnext --pio-install` hook fails before `pio run -t compiledb` can write `compile_commands.json`, so a fresh clone cannot create the database c-next reads framework headers from. |

The first three join existing cards [#1523](https://github.com/smeltworks/cnx/issues/1523)
and [#1524](https://github.com/smeltworks/cnx/issues/1524), which already
identified the missing interop axis from the tooling side; the repros here are
the concrete failing cells those asked for. All five are cross-linked.

**Do not work G1 until v0.4.1 ships.** Whether `tickSource` keeps its `uint32_t`
spelling is a genuine design choice, but it should be made against a transpiler
that reports the mismatch rather than one that passes it to gcc. The wrapper
workaround is verified to clear G1 in both languages on real ESP32 builds
(C++ alone; C-Next needs the G2 shim as well) — it is not committed, because
this branch is a record of gaps rather than of workarounds.

The C++ examples never pass through c-next, so c-next#1619 can only catch G1 for
C-Next consumers. The library needs its own answer either way: a wrapper in
each sketch, or an optional Arduino clock adapter that keeps the declaration to
one line (`tick: ArduinoClock.millis`). That choice is still parked until
v0.4.1.

G2 is an Arduino-core matter. G5 was addressed on `main` by #10 (a tick declares
its range). G6's polling requirement is documented there, at once per *half*
the tick's range rather than once per 2^32.

---

## G1 — `tick: millis` does not type-check on ESP32

**Severity: blocking.** This is the README's headline usage, and it fails on one
of the three boards with hardware on hand, in **both** languages.

`tickSource` generates `typedef uint32_t (*ElapsedTime__tickSource_fp)(void);`,
while every Arduino core declares `unsigned long millis()`. Whether those agree
is target-dependent:

| Target | `__UINT32_TYPE__` | `tick: millis` |
| --- | --- | --- |
| AVR | `long unsigned int` | compiles |
| ARM (Teensy, STM32) | `long unsigned int` | compiles |
| **Xtensa (ESP32), gcc 8.4.0** | `unsigned int` | **fails** |
| Xtensa (ESP32), gcc 13.2.0 / 14.2.0 | `long unsigned int` | compiles |

So AVR, ARM and Teensy pass by coincidence, not by design — the library's
contract is with a fixed-width type, the Arduino contract is with
`unsigned long`, and those happen to be the same type on some targets.

C (transpiled from `.cnx`) — a warning, which would have linked:

    src/main.c:20:62: warning: initialization of 'uint32_t (*)(void)'
    {aka 'unsigned int (*)(void)'} from incompatible pointer type
    'long unsigned int (*)()' [-Wincompatible-pointer-types]

C++ — a hard error:

    src/main.cpp:7:75: error: invalid conversion from 'long unsigned int (*)()'
    to 'ElapsedTime__tickSource_fp' {aka 'unsigned int (*)()'} [-fpermissive]

For the fix pass: a wrapper resolves it but costs the one-line declaration that
is the library's whole selling point, on the most popular 32-bit target.

**Update 2026-10-18 — it depends on the toolchain, not only the target.** The
failing row is xtensa gcc 8.4.0, which PlatformIO's `espressif32` platform
(6.12.0) uses for the Arduino core (arduino-esp32 2.0.17). The same platform
builds ESP-IDF with xtensa gcc 14.2.0, where `__UINT32_TYPE__` is
`long unsigned int`, as it is in 13.2.0. With those toolchains `tick: millis`
compiles. So G1 reproduces on the gcc 8.4 Arduino core, not on ESP32 as such.
A newer toolchain clears it, but the library cannot depend on which one a user
has.

## G2 — a `.cnx` main cannot supply `setup`/`loop` on ESP32

**Severity: blocking for the C-Next variant, ESP32 only.** Verified by reading
each core's header, not inferred:

| Core | `setup`/`loop` declared | Inside `extern "C"`? | Links from C |
| --- | --- | --- | --- |
| AVR | `Arduino.h` 154–155 | yes (block 35–227) | yes |
| STM32duino | `api/Common.h` 139–140 | yes (block 24–143) | yes |
| Teensy | — | yes (empirically) | yes |
| **ESP32** | `Arduino.h` 149–150 | **no** (block is 168–182) | **no** |

ESP32 declares them under a bare `#ifdef __cplusplus`, *before* its `extern "C"`
block, so they carry C++ linkage and the generated C cannot satisfy them:

    undefined reference to `setup()'
    undefined reference to `loop()'

An Arduino-core inconsistency rather than a library defect — but the library
ships the examples, so it needs an answer, most likely a documented C++ shim
forwarding `setup`/`loop` into the `.cnx`.

## G3 — transpiling a consumer rewrites the library's header to the wrong place

**Severity: papercut.** An example's `cnext.config.json` governs where the
*dependency's* generated header lands. Transpiling any example regenerates
elapsedTime too and drops `ElapsedTime.h` into the library's `src/`, beside the
canonical copy in `include/`:

    Generated 4 output files:
      .../elapsedTime/src/ElapsedTime.c
      .../examples/<ex>/cnx/src/main.c
      .../elapsedTime/src/ElapsedTime.h     <-- stray; canonical is include/
      .../examples/<ex>/cnx/src/main.h

`git status` then shows `?? src/ElapsedTime.h` after building any example. Every
example build in this survey needed a manual `rm` afterwards.

**Update 2026-10-18 — worse than a papercut; filed as
[c-next#2151](https://github.com/smeltworks/cnx/issues/2151).** The consumer's
run also rewrites the library's `.c` in the library's tree, for the *consumer's*
target. That is invisible here only because elapsedTime's output is byte-identical
on all 13 targets. A library with an `atomic` member is not: a Cortex-M4 consumer
turned a host library's `.c` into one that includes `<cmsis_gcc.h>`, and the
library's own build then failed. The stray header also carries a different include
guard from the library's own.

## G4 — C-Next resolves Arduino and nothing else

**Severity: high friction, affects every non-Arduino platform.**

**Correction 2026-10-18 — the heading is wrong; this survey missed the compile
database.** Since v0.2.18 (2026-07-10, c-next PR #1104), c-next reads include
paths, defines and the compiler from the project's `compile_commands.json`. The
table described below (now `src/cnx/TRANSPILE/1-Analyze/StdlibFunctions.ts`) is
only a fallback. Re-checked on c-next `main` after `pio run -t compiledb`, with
no framework path in `cnext.config.json`: the STM32 HAL example resolves every
header. The ESP-IDF example does too once the xtensa toolchain is on `PATH`,
because PlatformIO writes the compiler's bare name into the database. What
survives of G4:

- **The database cannot be bootstrapped:**
  [c-next#2154](https://github.com/smeltworks/cnx/issues/2154). The
  `cnext --pio-install` hook fails before `pio run -t compiledb` writes the
  database. c-next's docs tell the reader to list every header directory in
  `include` instead.
- **`vTaskDelay` really is unresolvable, but not for the reason given below:**
  [c-next#2153](https://github.com/smeltworks/cnx/issues/2153). The missing
  `FreeRTOSConfig.h` was this config's fault; it lives in
  `components/freertos/config/include/freertos/`, which was never listed. With
  the database, an include cycle in ESP-IDF's FreeRTOS headers makes `task.h`
  read as empty.
- **`defines` is not a `cnext.config.json` key.** It exists only on the
  programmatic API, and unknown config keys are dropped silently
  ([c-next#1490](https://github.com/smeltworks/cnx/issues/1490), open).
  `-D` on the command line and the database's `-D` flags both reach the
  preprocessor, and the database's flags clear the HAL's device-select `#error`.
- **The machine-specific paths are moot through the database.** In a
  hand-written config only `~` expands; a `$HOME/...` entry is dropped.
- **The database's compiler is not found off `PATH`:**
  [c-next#2356](https://github.com/smeltworks/cnx/issues/2356) (2026-10-25).
  c-next takes the database's bare `xtensa-esp32-elf-g++` as written and says
  it "is not installed", while `pio project metadata` reports its absolute path.
  ARM and AVR escape only because a system `arm-none-eabi-gcc` and `avr-gcc`
  happen to be on `PATH`.

The original finding follows, as recorded on 2026-09-20.

C-Next carries a hardcoded table (`src/TRANSPILE/1-Analyze/StdlibFunctions.ts`)
mapping known C functions to headers. It covers the C standard library plus an
"Arduino framework" block (`millis`, `micros`, `pinMode`, `digitalWrite`, …).
Proof it is a built-in table rather than header parsing: with `Arduino.h` *not*
on the include path at all, `millis()` resolves and a made-up name does not.

Every other ecosystem — ESP-IDF, STM32 HAL, Pico SDK, Zephyr — is absent, so a
bare call fails:

    error[E0422]: function 'esp_timer_get_time' called before definition
    error[E0422]: 'esp_timer_get_time' is not declared in any included header

The escape hatch works but is heavy. C-Next runs a **real C preprocessor** over
included headers, so `cnext.config.json` must reproduce the build's include
path, which the platform's own build system normally computes:

```json
"include": [
    ".../components/esp_timer/include/",
    ".../components/freertos/FreeRTOS-Kernel/include/"
]
```

Even then, headers with deep chains fail to preprocess and fall back to raw
scanning — `FreeRTOS.h` could not find `FreeRTOSConfig.h`, and `vTaskDelay`
stayed unresolvable, so the ESP-IDF example had to drop it.

**Sub-finding — the include paths are machine-specific.** `cnext.config.json`
takes no variable expansion, so the ESP-IDF and STM32 examples in this tree
carry hardcoded absolute paths into `~/.platformio/packages/...`:

    "include": ["/home/linux/.platformio/packages/framework-stm32cubef4/..."]

They build here and nowhere else. Shipping a `.cnx` example for any non-Arduino
platform needs an answer to this — a variable, a PlatformIO-provided path, or
generating the config in a build hook.

**Sub-finding, worth its own look:** `cnext.config.json` accepts a `defines` map
(`ITranspilerConfig.defines`), but setting `{"STM32F401xE": true,
"USE_HAL_DRIVER": true}` did **not** suppress the HAL's device-select `#error`.
Either the key is not honoured on this path or more is needed; it transpiled
anyway via raw-content fallback. Not chased further in this pass.

## G5 — a tick narrower than 32 bits cannot be adapted, and fails silently

**Severity: the most serious design gap found.** The README says the library
takes "any `u32` tick" and "anything with a free-running `u32` counter". Any
source that wraps at less than 2^32 breaks it, and the natural adapter —
widening to u32 — is catastrophically and silently wrong.

`sinceTick`'s wrap correction assumes the counter wraps at exactly 2^32:

    return (4294967295 - base) + now + 1;

Feed it a counter that wraps at 2^16 and that branch fabricates an enormous
interval. Minimal reproduction (a u16 counter widened to u32, 65000 → 100,
i.e. 636 real ticks):

    actual elapsed:   636
    library reports:  4294902396
    WRONG

This is not hypothetical. FreeRTOS with `configUSE_16_BIT_TICKS=1` makes
`TickType_t` a `uint16_t`. More common still, raw hardware counters are
routinely narrower than 32 bits — AVR Timer1 is 16-bit, ARM SysTick is 24-bit —
and the README explicitly invites feeding the library "a counter", including
`DWT->CYCCNT`-style sources.

For the fix pass: the library either needs to know the tick's modulus, or needs
to reject/document narrower sources loudly. Silent 6.7-million-fold error is the
worst possible behaviour.

## G6 — "unbounded, automatic" carries an undocumented polling requirement

**Severity: documentation gap with a real failure behind it.** The README
promises elapsed time past 49.7 days needs "nothing extra", that folding is
"internal and automatic", and that there is nothing "for a user to forget".

There is: **the caller must poll at least once per 2^32 ticks.** `sinceTick` can
only measure distances below one wrap, so a longer gap between calls loses a
whole wrap undetectably. Reproduction — seed at 0, let 5,000,000,000 ms pass,
then read:

    real time elapsed: 5000000000
    library reports:   705032704
    shortfall:         4294967296

The suite's `elapsed_is_unbounded_without_any_call_from_the_user` passes because
it advances the clock in small steps; it does not cover a single jump wider than
a wrap. At microseconds the requirement is "poll every 71.6 minutes", which a
sleepy low-power design can easily miss — and `remaining()` exists specifically
to help size long sleeps.

---

## Not gaps — C-Next ergonomics worth folding into the AI reference

Hit while writing the `.cnx` examples. All are the transpiler steering toward
MISRA-clean code, and all were resolved by following its own help text:

- `while (true)` → `error[E0701]` (MISRA 14.4, condition must be a comparison),
  then `error[E0707]` on the workaround, with the right answer in the help:
  a dedicated `forever { }` construct, which emits `for (;;)`.
- A discarded return value → `error[E0708]` (ADR-070); write `(void) printf(...)`.
- C functions sometimes need an explicit `global.` prefix where a bare call is
  not resolvable.
- `i64` → `u32` truncation via bit extraction, `now[0, 32]`, generates exactly
  `(uint32_t)((now) & 0xFFFFFFFFULL)` — clean, and the right answer for the
  ESP-IDF adapter.
