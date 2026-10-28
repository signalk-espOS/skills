---
name: espos-memory-budget
description: "Diagnose out-of-memory failures on ESP32 firmware (espOS or any ESP-IDF project): why PSRAM appears to do nothing, why a classic ESP32 reports internal RAM that malloc cannot get, which pool TLS and the radio really draw from, how to size task stacks from measurement instead of guesswork, and what keeps an image inside its app slot."
---

# Diagnosing an ESP32 memory budget

Verified against ESP-IDF **v6.0.3**, October 2026. The Kconfig defaults and the
allocation sites below are the things most likely to change, so re-check them against
your own IDF before trusting a number.

## The first rule: measure regions, not totals

`heap_caps_get_free_size()` returning "plenty free" while an allocation fails is the
normal shape of these bugs. Two different mistakes hide behind a total:

```c
#include "esp_heap_caps.h"

heap_caps_get_free_size(MALLOC_CAP_INTERNAL | MALLOC_CAP_8BIT);          /* general */
heap_caps_get_largest_free_block(MALLOC_CAP_INTERNAL | MALLOC_CAP_8BIT); /* fragmentation */
heap_caps_get_free_size(MALLOC_CAP_DMA);                                 /* a SUBSET */
heap_caps_print_heap_info(MALLOC_CAP_DMA);                               /* per region */
```

* **A large total with a small largest block** is fragmentation. A caller wanting one
  contiguous buffer fails while the total looks healthy. `esp_bt_controller_init()`
  wants ~24 KB in one piece; it fails at 33 KB free / 16 KB largest and allocates
  nothing, so free size is identical either side of the call.
* **`MALLOC_CAP_DMA` is a subset of internal memory**, not a synonym. Quote the wrong
  one and you overstate the headroom available to whatever you are debugging.
* **`MALLOC_CAP_INTERNAL` alone counts IRAM on a classic ESP32.** The IRAM left over
  after code is heap there, but only 32-bit accesses may use it, so no buffer, stack or
  WiFi frame can come from it. A HALMET (classic ESP32) reported 50304 B "free
  internal" next to a `free_heap` of 18544 B with a low-water of 1980 B, about 31 KB
  that `malloc()` could never get. Its free-RAM alarms stayed quiet and the one that
  fired said "fragmented" for a heap that was simply running out. Count
  `MALLOC_CAP_INTERNAL | MALLOC_CAP_8BIT`, as espOS does since 0.19.1 (#229); on chips
  whose internal heap is all byte-addressable the numbers do not change.

`heap_caps_print_heap_info()` is what settles an argument: it prints each region's
size and free space. A knob that "does nothing" often turns out to have created a
pool that then got consumed: the same end state from a different cause. Comparing
totals cannot tell those apart; a region dump can.

## Instrument the boot, per stage

Wrap whatever your firmware's startup does and print between stages. This turns "it
runs out of memory" into a budget:

| stage | internal RAM |
|---|---|
| HTTP server | 25.2 KB |
| network layer | 5.4 KB |
| Signal K client | 21.2 KB |
| OTA | 9.6 KB |
| BLE (Bluedroid host + controller) | 58.2 KB |

That is a real measurement from an ESP32-C5 with WiFi, and it is the artefact worth
having: it says which subsystem to attack, and it makes "there is nothing left to
move" a statement you can defend instead of a suspicion.

## PSRAM: four traps

**1. Enabling PSRAM alone may leave all of it unused.**
`CONFIG_SPIRAM_MALLOC_ALWAYSINTERNAL` defaults to **16384**, so every allocation
smaller than 16 KB stays internal, and most allocations are smaller than that. A
board can report `Found 8MB PSRAM device`, add it to the heap, and still exhaust
internal RAM with 8.2 MB idle. Lower it or nothing moves; the C5 BLE gateway runs at
256, with `CONFIG_SPIRAM_MALLOC_RESERVE_INTERNAL=32768` kept for allocations that ask
for internal or DMA memory.

**2. `CONFIG_SPIRAM_TRY_ALLOCATE_WIFI_LWIP=y` can stop WiFi working entirely.**
It is the option that moves the largest single block (tens of KB of WiFi and lwIP
buffers) out of internal RAM, and on some parts the station then never associates:
`connecting to '<ssid>'` followed by disconnect reason 36 (`WIFI_REASON_STA_LEAVING`,
i.e. your own state machine giving up), repeating indefinitely. Measured, repeatably,
on an ESP32-C5. The likeliest reason is that the driver needs those buffers in
internal memory, but that part is inference. Treat the option as one to test on your
own silicon rather than as a general fact about PSRAM and DMA. **Rule out the radio
first:** on a shared WiFi/BLE radio, a BLE scan at a 50 % duty cycle (160 ms every
320 ms) produced exactly the same reason 36 with this option *off*, and the station
joined within seconds once scanning paused. Pause the scan until the station has an
address before blaming PSRAM.

**3. Task stacks do not go to PSRAM by default, but your own can.**
`xTaskCreate()` allocates internally. With `CONFIG_FREERTOS_TASK_CREATE_ALLOW_EXT_MEM`
(default y with SPIRAM on IDF 6, except on the original ESP32), `xTaskCreateWithCaps(..., MALLOC_CAP_SPIRAM |
MALLOC_CAP_8BIT)` puts a task's stack in PSRAM, and `vTaskDeleteWithCaps(NULL)` lets
it delete itself. The rule: **the task must never run while the cache is off**, and
any flash access switches it off: NVS *reads* as well as writes, OTA, partition
reads. Audit every call path, including library ones: the espOS BLE gateway task
looked flash-free until an HTTPS POST turned out to load and save a TLS pin in NVS on
the caller's task; the fix hands those NVS calls to a short-lived task with an
internal stack when the caller's is in PSRAM (`esp_ptr_external_ram(&local)`).
Moved this way on the C5: the gateway task (8 KB) and mDNS (`CONFIG_MDNS_TASK_CREATE_FROM_SPIRAM`).
Not movable: anything that writes NVS (Signal K client, httpd, Bluedroid), OTA, WiFi/lwIP,
and tasks that run arbitrary callbacks (event loop, timer service).

**4. `CONFIG_SPIRAM` is *not* a bootloader change** (at least on the C5).
`esp_psram_init()` is called from `esp_system/port/cpu_start.c` under
`CONFIG_SPIRAM_BOOT_INIT`, i.e. during application startup. The boot log proves it:
`Loaded app from partition` appears *before* `Found 8MB PSRAM device`. So enabling
PSRAM ships by OTA like any other build, and does not require a USB flash per device.
Verify on your target before relying on it: some parts do map PSRAM earlier.

Also worth knowing: on ESP32-C5 rev v1.0 IDF warns that PSRAM contents are not
encrypted, even with flash encryption on, and names TLS buffers as an example. The
espOS C5 gateway puts them there anyway (`CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC`: the PSRAM
is in-package, so reading it takes a decapped chip). Make that call deliberately for
a release build, and make any TLS memory pre-flight measure the pool mbedTLS actually
allocates from.

**On a part where code and data share one SRAM (the C5), IRAM is heap.** IDF puts
WiFi speed code in IRAM by default (`CONFIG_ESP_WIFI_IRAM_OPT`, `_RX_IRAM_OPT`,
`_EXTRA_IRAM_OPT`, `_SLP_IRAM_OPT`). Turning the four off, in the same change as
moving ~7 KB of the gateway's static buffers to PSRAM, took a BLE gateway from 34.7
to 66.3 KB of free internal RAM, low-water 15.8 → 25.5 KB; IDF's Kconfig help puts the
four at roughly 39 KB of IRAM. The cost is WiFi throughput a few KB every two seconds
never needs. Likewise the WiFi driver's static
RX buffers must be internal, but 6 (= `CONFIG_ESP_WIFI_RX_BA_WIN`) instead of 10 frees
6.4 KB.

## Which pool does TLS use?

`MALLOC_CAP_INTERNAL | MALLOC_CAP_8BIT`, not `MALLOC_CAP_DMA`.
`components/mbedtls/port/esp_mem.c` allocates with exactly those caps under
`CONFIG_MBEDTLS_INTERNAL_MEM_ALLOC`, which is the **default** choice. So when you are
deciding whether a handshake will fit, the internal-8bit figure is the relevant one;
the DMA subset matters to the radio and the network driver instead.

If you gate a handshake on free memory, gate it on the same pool mbedTLS will ask
from, and on the *largest block*, since a handshake needs a contiguous allocation.

## Shrinking task stacks: measure first

```
CONFIG_FREERTOS_USE_TRACE_FACILITY=y
```
```c
TaskHandle_t h = xTaskGetHandle("my_task");
/* high water = the SMALLEST free the stack ever had; ESP-IDF returns bytes */
size_t unused = uxTaskGetStackHighWaterMark(h);
```

Real numbers from one firmware: a task given 12288 B used **1488**, 12 %. Another
given 8192 used 5272, 64 %. You cannot guess which is which.

Three cautions that cost real time:

* **A measurement only covers the paths it exercised.** That 12 % task was measured
  against a plain-HTTP server, so mbedTLS, its deepest call path, was never entered.
  A high-water reading with the deepest path untaken does not show the stack is
  oversized; it shows the measurement was incomplete. Take the reading against the
  heaviest workload the build supports (a `wss` server if TLS is compiled in).
  The 8192 B task above that used 5272 B was the espOS Signal K stream task on a quiet
  run. A 112-minute run of the same build, with REST requests every 10 minutes and OTA
  attempts on other tasks, peaked at 6904 B, so the default went to 10240 (espOS #212).
  That run was still plain ws, with no TLS on the stream task, and the PR says so,
  which is why nothing was lowered from it. Measure long and busy, and write down what
  the run did not exercise.
* **Never shrink a stack and change memory configuration in the same flash.** A
  too-small stack corrupts rather than failing cleanly, and the flash mode, frequency
  and size live in the bootloader header, which esptool writes at flash time. A bad
  combination can leave the chip unable to boot far
  enough to serve USB-JTAG, needing a physical BOOT+RESET. One risky variable per
  flash, and have a known-good image ready.
* **If you make a stack size configurable, floor it above what you measured.** An
  option whose range permits a value below observed usage is a trap, not a choice.

## Smaller gotchas

* **`-Werror=format-truncation` fires on runtime values.** A `snprintf` into a small
  fixed buffer is rejected at compile time even when both arguments are runtime
  strings and truncation is only theoretical. Shorten the format, do not widen the
  buffer to silence it: the buffer size is usually a protocol limit.
* **A fresh `sdkconfig` resets `IDF_TARGET`.** Deleting `sdkconfig` and building
  silently selects the default target, and `idf.py` refuses to retarget over an
  existing one. When switching chips: `rm -f sdkconfig && rm -rf build`, then
  `set-target`. A stale `sdkconfig` is the usual reason an image comes out for the
  wrong chip, and a device will reject that image *after* a full OTA download.
* **A prebuilt library can lose a symbol at link time.** esp-zigbee-lib's coordinator
  build on IDF 6.0.3 failed to link: `libzboss_stack`'s Green Power cluster calls back
  into `libesp_zb_api`, which the linker has already passed under IDF 6's library
  order. `target_link_libraries(${COMPONENT_LIB} INTERFACE
  "-Wl,--undefined=zb_zcl_green_power_cluster_handler")` pulls the handler in up
  front. When a vendor `.a` reports an undefined
  symbol that `nm` shows it defines, try `--undefined=<symbol>` before reordering
  components.

## The app slot is a budget too

A device keeps the partition table it was first flashed with, so an image that outgrows
its app slot can never be installed over the air on boards already in the field. Keep
headroom from the first release:

* **Build for size.** IDF's default `-Og` costs about 12 % of the image against `-Os`
  (`CONFIG_COMPILER_OPTIMIZATION_SIZE`). An ESP32-C5 image went from 92.8 % to 81.7 % of
  a 4 MB table's 1728 KB slot. Keep `-Og` for a debug profile, where backtraces matter
  more.
* **Images grow in 64 KB pages.** 1.3 KB of new code pushed that C5 image over a page
  boundary and a 90 % budget check red. Fail CI on a budget (espOS uses 90 %) rather
  than on "does not fit", which is too late.
* **Size the slots for the largest table you will ever offer a chip**, and offer a
  smaller flash size only where the image stays under the budget there.

## When the answer is "it does not fit"

First check that it is: the ESP32-C5 was declared unable to run BLE + WiFi + a Signal K
client ("no configuration changes that"), and it ran (internal RAM 66 KB free) once
the levers above were pulled in order: PSRAM on with `ALWAYSINTERNAL` lowered, the
Bluedroid host heap to PSRAM (`CONFIG_BT_ALLOCATION_FROM_SPIRAM_FIRST`), TLS buffers to
PSRAM, static RX buffers trimmed, WiFi IRAM code back to flash, flash-free task
stacks to PSRAM, and the scan paused while the station associates. The original
verdict had been measured with PSRAM on but every allocation under 16 KB still
internal. Only after that:

Worth saying out loud, because the alternative is an open investigation nobody closes.
If the per-stage budget accounts for nearly all of internal RAM and every remaining
consumer is a task stack or a DMA buffer, the honest conclusion is a hardware verdict,
not a defect. Write the budget table down, say which combinations *do* work, and
record it where somebody choosing hardware will read it: a board's own docs and,
if you publish firmware, wherever your release metadata advertises targets.
