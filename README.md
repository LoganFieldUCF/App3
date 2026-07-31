# App3
https://wokwi.com/projects/468079615928883201

# Concurrency Diagram

<img width="677" height="729" alt="concurrencyDiagram" src="https://github.com/user-attachments/assets/8fe89f59-4fb9-4a42-b6d7-49284c867f7c" />

# Logic Analyzer

<img width="1070" height="844" alt="logicAnalyzer" src="https://github.com/user-attachments/assets/a9e7ea64-47ed-4d00-93bd-c0d2d5c0d6f9" />

# App 3 scaffold — Interrupts & bottom-half

Scaffold level: **~70% complete**.

## What's given

- Complete `IRAM_ATTR` ISR with debounce, scope pulse on GPIO 19
- Two bottom-half paths: binary semaphore AND direct task notification
- Live latency measurement (µs) printed on every press
- Wokwi diagram with button + indicator LED
- An **optional background load** (App 2's four tasks) behind one `#define`, so "measure latency under load" is a flag flip rather than a copy-paste

## Run modes (`WITH_LOAD`)

App 3 reports the same per-press latency fields in both modes; only the contention on Core 1 changes. The switch is one line at the top of `main.c`:

```c
#define WITH_LOAD 0   /* 0 = idle baseline, 1 = App 2's 4 tasks on Core 1 */
```

- `WITH_LOAD = 0` — **idle**. Only the two bottom-half tasks live on Core 1. This is your baseline `latency-max`.
- `WITH_LOAD = 1` — **loaded**. App 2's four periodic tasks (10/20/50/100 ms) run on Core 1 at the rate-monotonic ladder (15/10/5/2). They are a fixed load fixture — deterministic, peripheral-free compute (xorshift, FIR, CRC-32, forced-worst-case insertion sort) carried over from App 2 so the load is reproducible and Wokwi-safe.

**Priority geometry you must account for:** your bottom-half tasks sit at priority **12**. Load Task A is **15**, so it outranks them; load Tasks B/C/D (10/5/2) do not. Under load, Task A can therefore delay a wake while B/C/D cannot preempt your bottom half. That asymmetry is the result you explain in the analysis.

## What you do

1. **Theme rename** — `YOURTHEME` and customize the log messages
2. **Run >= 50 presses, idle** (`WITH_LOAD 0`). Record `latency-max` for both paths.
3. **Flip to `WITH_LOAD 1`**, rebuild, and run >= 50 presses again. Re-record both paths. Confirm the four load tasks are live (their heartbeat counters climb).
4. **Induce a failure** — pick ONE and document the symptom:
   - Remove `portYIELD_FROM_ISR(higher_woken)` → notification is delivered, but the task doesn't run until the next tick
   - Remove `IRAM_ATTR` → first-press latency on cold cache spikes 10-100x
   - Replace `xSemaphoreGiveFromISR` with `xSemaphoreGive` → undefined behavior; system may crash
5. **Defend in README** (see prompts below)

## Capturing latency with Wokwi's logic analyzer

1. In Wokwi, click the "+" near the chip, add a "Logic Analyzer".
2. Connect channel 0 to GPIO 18 (button), channel 1 to GPIO 19 (ISR pulse).
3. Run, click the button N times.
4. Click the logic-analyzer to download a VCD file.
5. Open in PulseView / sigrok or Wokwi's built-in viewer.
6. Measure: time from button-low edge to GPIO-19 rising edge = total interrupt response time (HW latency + your debounce gate + the ISR prologue).
7. Time from GPIO-19 falling edge to the next `[sem]` or `[notif]` log line = bottom-half wake latency. This is the more interesting number, and it's the one that moves when you flip `WITH_LOAD`.

## Engineering analysis (README, graded)

1. **What's in your ISR? What's NOT?** List every line. Defend each (or remove it).
  In IRS:
    int64_t now = esp_time_get_time(); // records timestamps to measure latency
    gpio_set_level(ISR_PULSE_GPIO, 1); // sets gpio 19 high to make a pulse captured by the logic analyzer
    isr_entry_time_us = now; // saves the interrupt entry time for bottom half tasks
    presses_obvserved++; // counts the total button presses
    BaseType_t higher_woken = pdFALSE; // required by FreeRTOS to determine if high priority tasks should run
    xSemaphoreGiveFromISR(btn_sem, &higher_woken); // signals the semaphore using FreeRTOS function
    vTaskNotifyGiveFromISR(task_notif_handle, &higher_woken); // signals the task notification using FreeRTOS function
    gpio_set_level(ISR_PULSE_GPIO, 0); // sets gpio 19 low to complete logic analyzer
    portYIELD_FROM_ISR(higher_woken); // switches if a bottom half task gets awakened
  NOT in ISR:
    printf()
    ESP_LOGI()
    Dynamic memory allocation
    Delays
    Blocking FreeRTOS calls
    Mutexes
    All of these operations are performed by bottom half tasks after completing the interrupt to keep ISR short.
2. **Binary semaphore vs direct task notification** — quote your measured latency numbers. Which is faster? Why?
    Binary Semaphore: Idle: 11us Loaded: 18us
    Direct Task Notification: Idle: 8us Loaded: 14us
    The direct task notification produced lower wake up latency because they entirely skip memory allocation and write directly onto the task's control block.
3. **Latency under load** — quote idle (`WITH_LOAD 0`) vs loaded (`WITH_LOAD 1`) numbers. By what factor does latency increase? Use the priority geometry above (Task A at 15 outranks your bottom half at 12) to explain *which* load task is responsible for the worst-case increase, and why B/C/D are not.
    Idle: 8us
    Loaded: 14us
    Factor = 14/8 = 1.75X
    The latency increase occurs due to Load Task A because it executes at a higher priority, therefore it can delay the other bottom half tasks.
4. **Induced failure** — name the rule you broke, the predicted symptom, the observed symptom, and how they match (or don't).
    portYIELD_FROM_ISR(higher_woken); was the broken rule. My prediction was that it would not switch to the awakened task after signaling it. I observed that the latency increased because the bottom half task did not execute right after the interrupt, just as predicted.

## Common pitfalls

- **Calling `printf` inside the ISR.** `printf` takes a UART mutex. Mutex from ISR = undefined behavior. The scaffold puts logging in the BOTTOM-HALF tasks for a reason.
- **Forgetting `IRAM_ATTR`.** The first interrupt after a long quiescent period has to load the ISR from flash. That's ~10s of µs of cache fill on top of your nominal latency. With `IRAM_ATTR`, the ISR is in always-on internal RAM.
- **Debounce too short.** A clean push-button bounces for 1–10 ms typically. Wokwi's simulated button is clean, but if you wire a real button, drop `DEBOUNCE_US` to something like 10000 µs.
- **Editing the load-task bodies.** Under `WITH_LOAD 1` the four tasks are a fixture, not the assignment. You're timing your ISR path, so leave their bodies alone; tune only the `*_ITERS`/`*_N`/`*_LEN` knobs if you want a heavier or lighter load.
- **Both bottom-half tasks racing on `latency_max_*`.** This is fine for the scaffold (32-bit reads are atomic, and the max-update is benign-racy). In production you'd use atomics or a mutex — that's App 6's lesson.

## Setup in Wokwi

Same shape as App 1. In a fresh Wokwi ESP-IDF project (`https://wokwi.com/projects/new/esp32-s3`):

1. Replace `diagram.json`, `wokwi.toml`, and `main/CMakeLists.txt` with this folder's versions. (App 3 has no `sdkconfig.defaults` &mdash; uses IDF defaults.)
2. Place this folder's `main.c` at `main/main.c` (delete Wokwi's `main/src/`), **or** leave `main/src/main.c` and edit `main/CMakeLists.txt` to use `SRCS "src/main.c"` + `INCLUDE_DIRS "src"`.
3. Confirm `wokwi.toml`'s `firmware` / `elf` paths match `app3_interrupts` (the `project(...)` name in `CMakeLists.txt`).
4. Click &#9654;.

**No web page in App 3.** All output is on the serial monitor; visual feedback is the yellow ISR-pulse LED on GPIO 19. Turning on the background load (`WITH_LOAD 1`) needs no extra components and no Wi-Fi &mdash; the four tasks are pure compute on Core 1. The Wokwi logic-analyzer instructions above are how you capture the timing numbers.

### Build locally with ESP-IDF instead

```bash
. $HOME/esp/esp-idf/export.sh
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

To build the loaded configuration from the command line without editing the file, override the flag at configure time:

```bash
idf.py build -DWITH_LOAD=1     # or set it in main.c
```

## Honor code

AI to explain ESP-IDF GPIO ISR mechanics &mdash; fine, cite. AI to write the analysis for you &mdash; no. The latency numbers are yours; the interpretation is yours.

Utilized ChatGPT to generate an image of a concurrency diagram. I provided the code and in depth instructions on how it should look. This was simply to avoid the complications of performing all of the charting by hand.
https://chatgpt.com/share/6a4136f4-73d4-83ea-ab96-8ba1c8e0c3d1
