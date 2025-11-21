# SenseCAP Watcher – OpenAI Realtime Demo


Based of the openai-realtime software from the official repository https://github.com/Seeed-Studio/SenseCAP-Watcher-Firmware

This repository contains the ESP-IDF firmware that powers the **SenseCAP Watcher** demo unit shown in the OpenAI Realtime video. The app pairs the Watcher’s camera, microphone, dial, and speaker with OpenAI’s Realtime API to deliver hands-free multimodal assistance:

- **Vision** – capture a still image and ask for a spoken description.
- **Vision + Voice** – snap a picture, describe what you care about, then listen to the response.
- **Voice Chat** – hold a button or tap the dial to have a back-and-forth conversation.
- **Smart Timers** – request timers in natural language (“make sure the steak rests for 8 minutes”) and manage them directly from the device UI.
- **Clear Context** – reset the current conversation and start fresh.

The circular touchscreen shows a simplified UI: a glowing “face” that reflects the current audio state plus a bottom tray of square action buttons that can be browsed with the dial or encoder.

---

## Quick Start

### 1. Prerequisites

- [ESP-IDF v5.2](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/get-started/index.html) installed and added to your shell.
- GNU Make/CMake/Ninja (bundled with ESP-IDF).
- Python 3.11+ with the ESP-IDF virtualenv activated.
- An ESP32-S3 SenseCAP Watcher devkit connected over USB-C.

### 2. Fetch Dependencies

```bash
git clone https://github.com/<your-account>/SenseCAP-Watcher-Firmware.git
cd SenseCAP-Watcher-Firmware/examples/openai-realtime
git submodule update --init --recursive
```

If you downloaded the `openai-realtime.zip` bundle, be sure to extract it inside `examples/` and run the command above once so that lvgl, rlottie, and other components are available.

### 3. Configure (optional)

```bash
idf.py menuconfig
```

Most defaults are ready for the SenseCAP Watcher board. Only tweak if you need a different Wi-Fi country code, UART log level, etc.

### 4. Build & Flash

```bash
idf.py build
idf.py flash monitor
```

> **Tip:** The build pulls several managed components (button, knob, led_strip, esp_codec_dev, etc.). The repository already includes pinned copies under `managed_components/` so you can compile offline.

### 5. Provision Wi-Fi & OpenAI Key

After the device reboots you should see a serial prompt. Use the built-in commands to store credentials:

```bash
openai_api -k sk-your-openai-api-key
wifi_sta  -s YourSSID -p YourPassword
```

Once connected, the Watcher automatically negotiates with the OpenAI Realtime endpoint specified in `sdkconfig`.

---

## UI Overview

The new UI is intentionally minimal to fit the round display:

- A glowing **face** in the center reflects the state:
  - Blue spinner = listening.
  - Orange flashes = speaking.
  - Red = error (network, API failure, etc.).
- A **tray** of square buttons along the bottom, each with a custom icon and label. Rotate the dial or encoder to move focus; press to activate.
- A slim **status line** above the tray shows contextual hints (“Describe what to ask about the photo”, “Recording… release to send”, etc.).

The tray scrolls with snap-to-center behavior so the first and last cards are fully visible.

---

## Built-in Actions

- **Vision / Vision + Voice** – Capture a frame, ask a question, and hear the result.
- **Voice Chat** – One-press record and send; tap again to process.
- **Timers** – Natural-language timers with live countdowns in the Timers card (e.g., “set a timer for 30 seconds”). Cancel from the inline ✕ button.
- **Reminders** – Schedule a reminder with “remind me to take cookies out in 30 seconds” or “set a reminder for 6 45 a.m.”. The Reminders card shows the countdown and lets you cancel.
- **Recorder** – Capture a short note; the device summarizes it and shows the last two summaries in the Notes card. “Clear Notes” wipes local summaries.
- **Clear Context** – Clears the running chat context.

> Tip: The preview panel above the tray switches per card: Timers shows countdowns, Reminders shows upcoming reminders, Recorder shows saved notes, others show activity/status.

---

## Adding Your Own Cards

1. Edit the `menu_items` array in `src/ui/ui.cpp`.
2. Provide a title, description (shown in the status line), click handler, and accent color.
3. Update `build_menu_icon()` with a new drawing routine or reuse an existing icon snippet.
4. Rebuild + flash.

Each handler can trigger anything—start a FreeRTOS task, call an OpenAI API function, toggle a GPIO, etc.

### Custom Icons

- Use the [LVGL Image Converter](https://lvgl.io/tools/imageconverter) to export bitmap icons as **C array** descriptors. A 1-bit alpha or ARGB (2/4/8 bit) image at `~64×64 px` fits cleanly inside the 80×80 cards.
- Set the symbol name to `ui_icon_<something>` so it matches the `LV_IMG_DECLARE(ui_icon_<something>);` usage in `src/ui/ui.cpp`.
- Drop the generated `.c` file into `src/ui/icons/` and add it to `UI_SRCS` in `src/CMakeLists.txt`.
- Keep colors high-contrast (white on transparent works best) so the rounded square background provides the accent color.
- If you add multiple icons, update `build_menu_icon()` to point each button to the appropriate descriptor.

---

## Manual Flashing

If you prefer to burn prebuilt binaries:

```bash
cd firmware
pip install --upgrade esptool
esptool.py --chip esp32s3 -b 460800 --before default_reset --after hard_reset write_flash \
  --flash_mode dio --flash_size 32MB --flash_freq 80m \
  0x0 bootloader/bootloader.bin \
  0x8000 partition_table/partition-table.bin \
  0x110000 openai-realtime.bin
```

> ⚠️ **Do not erase the entire flash**—it contains production data such as factory calibration and MAC addresses.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Build fails with “path field… does not point to a directory” | Ensure `managed_components/` contains the `espressif__button`, `espressif__knob`, `espressif__led_strip`, etc. directories. Copy them from the factory firmware bundle if needed. |
| UI buttons clipped on the sides | Rotate the dial once so the card snaps into the middle. The layout uses LVGL’s snap scrolling—first and last cards will recentre automatically. |
| No icons on the tray | The custom icons are drawn with LVGL primitives; if you customized colors, ensure they contrast against the button background. |
| No audio from the speaker | Confirm `esp_codec_dev` initializes by checking logs. The Watcher mutes audio when the hardware button is held; release it to hear the response. |

---

## License

Unless otherwise noted, code is licensed under the terms in `LICENSE`. Individual components under `components/` and `managed_components/` retain their original licenses. See each directory for details.
