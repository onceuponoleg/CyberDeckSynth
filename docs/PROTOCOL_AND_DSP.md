# Архитектура прошивки, Прямой UART и Pure Mono DSP (V5.2 Dual-ESP32)

## 1. Структура проекта PlatformIO (Единый репозиторий)

Обе платы прошиваются из одного репозитория (поочередным подключением кабеля Type-C напрямую в `ESP #1` или `ESP #2` при снятом плоском дне и выключенном тумблере `PWR`). Протокол и распиновка синхронизированы через общие заголовки в `include/`.

```text
cyber-deck-one/
├── platformio.ini                 # 2 окружения: [env:esp1_brain_sound], [env:esp2_warp_fx]
├── include/
│   ├── shared_protocol.h          # Структуры PacketBrainToFx и PacketFxToBrain (по 16 байт)
│   ├── pinout_config.h            # Единые макросы пинов для обеих плат и PCF8575 (V5.2)
│   └── ghost_samples.h            # ~20 предгенерированных радио-сэмплов (PROGMEM Flash)
├── src/
│   ├── esp1_brain_sound/main.cpp  # HUD 2.9" SPI, PCF8575, 7x EC11, Clock/Gen, Grids+Moog+Synth
│   └── esp2_warp_fx/main.cpp      # I2S0 Full-Duplex Master -> MAX98357A, Mono FX, 4x ADC1, 5x EC11
├── tools/
│   └── wav_to_header.py           # Конвертер ~20 WAV-фрагментов в include/ghost_samples.h
├── cad/
│   └── cyber_deck_slim_acoustic.scad
└── docs/                          # Документация проекта
```

---

## 2. Распределение 4 ядер FreeRTOS (Жёсткая изоляция звука)

Аудио-движок работает на фиксированной частоте **`44 100 Гц`**, DMA-буфер **`64` сэмпла** (задержка `~1.45 мс`). На Ядре 1 обеих плат запрещены `Serial`, `delay()`, I2C и блокирующие мьютексы — обмен с Ядром 0 идет через lock-free атомарные структуры.

| Плата | Ядро 0 (`Control / IO / Comm`, Приоритет 1) | Ядро 1 (`Audio / DSP`, Приоритет 5) |
| :--- | :--- | :--- |
| **`ESP #1`**<br>**`BRAIN & SOUND`** | • Аппаратный `esp_timer` (`2500 Гц`) на 7 энкодеров `EC11` (таблица Грея, 0 пропусков)<br>• Опрос `PCF8575` (`100 кГц` I2C, таймаут `6 мс`, авто-сброс шины) — 8 кнопок + 6 тумблеров (включая `FXOFF` и `DRV`)<br>• Экран `~2.9"` SPI (DMA SPI, `30 FPS`, Event-Driven всплывающие шкалы)<br>• Master Clock, `Harmonic Lock`, Perlin / Euclid / `MI Marbles (Deja Vu)`<br>• Детектор `usb_serial_jtag_is_connected()` + прямой `UART` с `#2` | • Непрерывный аудио-рендеринг (`DaisySP` / локальное DSP-ядро):<br>  - `Slot L` (Моно): `MI Grids` (Kick/Snare/Hat + Punch + Fill) + 24dB Moog Bass + Wavefolder/FM `Growl`<br>  - `Slot R` (Моно): Duo Synth (`Call & Response`, 2-3 голоса: SuperSaw Unison / Retro Pad / Cyber FM) + `Swell` + `Oct-/Oct+`<br>• Расчет огибающей `kick_gate` (`0..255`)<br>• Вывод в `I2S0 Slave TX` |
| **`ESP #2`**<br>**`WARP FX & OUTPUT`** | • Аппаратный `esp_timer` (`2500 Гц`) на 5 энкодеров (`SHIM, ECHO, PUMP, GHOST, TIMB`)<br>• Опрос 4 красных кнопок (`STUT/STOP/RISE/VOX`) и тумблера `SPACE`<br>• Чтение 4 потенциометров `ADC1` (`HIGH/MID/LOW` + `VOLUME`) с EMA-сглаживанием<br>• Прямой `UART` с `#1` (пересылка `TIMB`, событий HUD и 8 полос `VU`)<br>• Декодер ~20 Flash-сэмплов `Ghost` | • `I2S0 Full-Duplex Master`: прием Dual-Mono `L/R` от `#1`<br>• Сайдчейн канала `R` + `Ghost` от `kick_gate`<br>• **Pure Mono FX** (нагрузка на 45% ниже стерео): `Mono Shimmer +12/+19`, `Dub Tape Echo`, `Mono Chorus`, `Stutter / Tape Stop / Auto-Riser`<br>• **Digital Diode Drive (`DRV`)** -> 3-полосный EQ -> защитный `HPF 90 Гц` -> Soft Limiter -> логарифмический `VOLUME` (с лимитом `20%` при `USB Safe Mode`)<br>• Вывод `L = R = mono_out` в `MAX98357A` |

---

## 3. Аудио-шина `I2S0` (Dual-Mono Stem Input -> Pure Mono Output)

Вся цифровая коммутация звука использует **один** аппаратный контроллер `I2S0` в режиме **Standard Full-Duplex**:

* **`ESP #2` (`I2S_NUM_0`, Standard Full-Duplex Master):**
  * Генерирует общий кварцевый такт `BCK` (`GPIO 41`) и `WS` (`GPIO 42`) на частоте `44.1 кГц` (`16-bit` или `32-bit` stereo frame), раздавая его сразу на `ESP #1` и `MAX98357A`.
  * По линии `DIN` (`GPIO 39`) принимает раздельные моно-стволы от `ESP #1`:
    * `Slot L` = **Драм + Бас** (сухой фундамент: не попадает в Shimmer).
    * `Slot R` = **Синтезатор** (подается в сайдчейн, Mono Chorus, Dub Tape Echo и Shimmer `+12/+19`).
  * По линии `DOUT` (`GPIO 40`) выдает готовый моно-микс (одинаковый сэмпл в оба полуфрейма `L` и `R`) прямо на `MAX98357A`.
* **`ESP #1` (`I2S_NUM_0`, Standard Slave TX):**
  * Синхронизируется по внешним `BCK`/`WS` от `ESP #2` и упаковывает `Drums+Bass` в `Slot L`, а `Duo Synth` в `Slot R`.

---

## 4. Прямой дуплексный протокол UART (`921 600` бод, `0.17 мс`)

Прямое соединение `ESP #1 <-> ESP #2` (8N1, `921600` бод). Передача 16-байтного кадра занимает **`0.17 мс`** через аппаратный FIFO на Ядре 0 (`0%` нагрузки на звук).

### Структуры кадров (`include/shared_protocol.h`, V5.2)

```cpp
#pragma pack(push, 1)

// Направление 1: ESP #1 (Brain & Sound) -> ESP #2 (Warp FX & Output)
// Отправляется каждые 5 мс (200 Гц) + немедленно при ударе бочки / смене тумблеров
struct PacketBrainToFx {
    uint8_t magic;        // 0xA5 — маркер начала кадра
    uint8_t seq;          // Счетчик кадров (0..255)
    uint8_t sys_flags;    // bit0: BootDone, bit1: USB_PC_SafeMode, bit2: MuteAll,
                          // bit3: FxKill (PCF8575 P16), bit4: DriveOn (PCF8575 P17)
    uint8_t bpm;          // 60..160 BPM
    uint8_t step_idx;     // Текущий шаг секвенсора (0..63)
    uint8_t genre_mood;   // bit0-1: Genre (0:Techno, 1:Synth, 2:LoFi), bit2: Mood (0:Dark, 1:Bright)
    uint8_t root_scale;   // low4: Root Note (0..11), high4: Scale ID (0..7)
    uint8_t chord_degree; // Текущая ступень аккорда (0..6) — для подстройки Shimmer +19
    uint8_t kick_gate;    // 0..255: мгновенная амплитуда бочки (триггер сайдчейна)
    uint8_t sound_flags;  // bit0: DrumsKill, bit1: BeatSolo, bit2: FillActive,
                          // bit3: OctDown, bit4: OctUp, bit5: HoldArp
    uint8_t seed_counter; // Инкремент при нажатии SEED
    uint8_t reserved[4];  // Резерв под расширение
    uint8_t crc8;         // Контрольная сумма (полином 0x07) по первым 15 байтам
};

// Направление 2: ESP #2 (Warp FX & Output) -> ESP #1 (Brain & Sound)
// Отправляется 60 раз/сек (для VU-баров) + немедленно при вращении TIMB / FX / EQ / VOLUME
struct PacketFxToBrain {
    uint8_t magic;        // 0x5A — маркер начала кадра
    uint8_t seq;          // Счетчик кадров (0..255)
    uint8_t timbre_val;   // Ручка TIMB (0..255) -> управляет тембром/Swell синта на ESP #1
    uint8_t fx_flags;     // bit0: Stutter, bit1: TapeStop, bit2: Riser, bit3: Vox,
                          // bit4-5: SpaceMode (0..2)
    uint8_t hud_param_id; // ID измененного параметра (TIMB / FX / EQ / VOLUME, 0 = нет)
    uint8_t hud_param_val;// Значение (0..255) для всплывающего окна на экране 2.9"
    uint8_t vu_bars[8];   // 8 полос уровня/спектра (0..255) для анализатора на экране 2.9"
    uint8_t reserved;     // Резерв
    uint8_t crc8;         // Контрольная сумма (полином 0x07) по первым 15 байтам
};

#pragma pack(pop)
```

---

## 5. Карта памяти Flash и PSRAM на `ESP #2` (Pure Mono FX + Ghost)

| Ресурс памяти `ESP #2` | Общий объем | Распределение под задачи |
| :--- | :---: | :--- |
| **Внутренняя Flash** | `16 МБ` | • Прошивка + DSP-ядро: `~1.1 МБ`<br>• **Банк `ghost_samples.h` (~20 радио-сэмплов `16 кГц` моно, `16-bit PCM` или `4-bit IMA-ADPCM`):** `0.4 - 1.8 МБ`<br>• Свободный резерв: `>12 МБ` |
| **Внешняя Octal PSRAM** | `8 МБ` | • Моно-буфер `STUTTER / TAPE STOP` (`4 с` моно @ `44.1 кГц` float): `~700 КБ`<br>• Буфер `DUB TAPE ECHO` (`2 с` моно @ `44.1 кГц` float): `~350 КБ`<br>• Буфер `MONO SHIMMER REVERB` (`+12/+19`) + `MONO CHORUS`: `~1.0 МБ`<br>• Свободный резерв: `~5.9 МБ` |
| **Внутренняя SRAM** | `512 КБ` | • DMA-буферы `I2S0` (RX/TX), 3-полосный EQ, Digital Drive, Limiter, стек FreeRTOS |
