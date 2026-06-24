# JC3636K518 (K5 knob) - dokumentacja sprzętu i porting z K718C

Ta gałąź (`k518-alpha`) to próba uruchomienia firmware na **Guition JC3636K518**
(seria K5 knob) - bliźniaczej płytce klona Waveshare ESP32-S3-Knob-Touch-LCD-1.8.
Bazowy target to JC3636K718C (patrz `HARDWARE.md` w korzeniu projektu).

Źródło danych: oficjalne demo producenta `JC3636K518CN_knob_EN.zip` (chmura
jczn1688) - pliki w `docs/k518/demo/` oraz **schemat** w `docs/k518/schematic/`
(wersja V2.0). Płytka: ESP32-S3R8, 8 MB PSRAM (octal), 16 MB Flash, okrągły
1.8" 360x360, sterownik **ST77916** (QSPI), touch **CST816**, enkoder obrotowy,
DAC **PCM5100A**, **PDM** mikrofon, slot TF (SD_MMC), silnik haptyczny DRV2605.

> WAŻNE: init ST77916 (`demo/scr_st77916.h`) jest **bajt w bajt identyczny** jak
> na K718 - cała `init_sequence` w `base/core.yaml` zostaje bez zmian
> (invert ON / cmd 0x21, RGB565, 360x360).

> PUŁAPKA: piny audio w `demo/pincfg.h` to wadliwy szablon (BCK 18 / WS 16 /
> DO 17 kolidują z magistralą QSPI ekranu). **Nie używać ich.** Prawdziwe piny
> audio (i wszystkie inne poniżej) pochodzą ze schematu V2.0, który jest
> rozstrzygający.

## Mapa pinów - K518 vs K718

| Funkcja | K718C (main/beta) | **K518 (ta gałąź)** |
|---|---|---|
| QSPI SCK (clk) | 11 | **13** |
| QSPI CS | 12 | **14** |
| QSPI D0 | 13 | **15** |
| QSPI D1 | 14 | **16** |
| QSPI D2 | 15 | **17** |
| QSPI D3 | 16 | **18** |
| LCD RST | 17 | **21** |
| Podświetlenie BLK | 21 | **47** |
| Touch SDA | 9 | **11** |
| Touch SCL | 10 | **12** |
| Touch INT | 7 | **9** |
| Touch RST | 8 | **10** |
| Enkoder A | 2 | **8** |
| Enkoder B | 1 | **7** |
| DAC BCK | 3 | **39** |
| DAC WS / LRCK | 45 | **40** |
| DAC DO / DIN | 42 | **41** |
| DAC MUTE (XSMT) | 46 (trzymać HIGH) | brak na GPIO (stały/pull) |
| Mic clk (PDM SCK) | 5 | **45** |
| Mic data (PDM DATA) | 4 | **46** |
| Bateria ADC | 6 | **1** |
| LED ring (WS2812) | 0 (13 diod) | **brak (nie ma sprzętu)** |
| SD_MMC CLK | 39 | **4** |
| SD_MMC CMD | 38 | **3** |
| SD_MMC D0 | 40 | **5** |
| SD_MMC D1 | 41 | **6** |
| SD_MMC D2 | 48 | **42** |
| SD_MMC D3 | 47 | **2** |

(Mapowanie GPIO odczytane z arkusza `schematic/2_ESP32S3-R8.png`.)

## Różnice, które zmieniają zakres "wszystko działa"

1. **Brak pierścienia LED (WS2812).** Na schemacie K518 nie ma sieci LED.
   Na K718 ring siedzi na GPIO0; na K518 GPIO0 pełni inną rolę (patrz niżej).
   -> blok `light: esp32_rmt_led_strip` + skrypt `ring_update` i wszystkie
   reakcje ringu (asystent/timer/alarm/głośność) trzeba **usunąć/wyłączyć**.

2. **Przełącznik I2S CH445P sterowany GPIO0 (`I2S_SWITCH_IN`).** Magistrala I2S
   ESP32 jest multipleksowana przez układ CH445P (arkusz `schematic/5_DAC.png`).
   DAC dostaje sygnał tylko, gdy GPIO0 jest ustawiony poprawnie. GPIO0 to też pin
   BOOT (strapping). To najtrudniejszy element portu - do dopracowania na żywo
   (najpewniej `output: gpio` + ustawienie stanu na starcie).

3. **Mute DAC (XSMT)** nie jest na oczywistym GPIO (stały poziom / pull) -
   prawdopodobnie nie trzeba sterować mute z firmware (inaczej niż K718).

4. **Inne piny mic/DAC** - sam VA jest przenośny, ale piny remapować wg tabeli,
   nie kopiować 1:1.

Dodatkowo K518 ma silnik haptyczny DRV2605 (I2C) i drugi ESP32 - nieużywane
przez ten firmware.

## Plan portu

Fork `base/core.yaml` -> `base/core-k518.yaml`, zachowując te same `id:`
(`main_display`, `backlight_light`, `i2s_out`/`i2s_in`, `va`, sensory toucha
i knoba), przepiąć piny z tabeli, **wyrzucić** `ring_light` + skrypty ringu,
**dodać** sterowanie przełącznikiem I2S na GPIO0. `base/screens/*` i `base/watchfaces/*`
są pin-agnostyczne i przechodzą bez zmian (poza ewentualnymi odwołaniami do ringu).

## Materiały w tym katalogu

- `demo/` - pliki źródłowe z dema producenta (pincfg.h, scr_st77916.h, knob*).
- `schematic/` - schemat V2.0 (LCD&POWER, ESP32S3-R8, ESP32-CHIP, OTHER, DAC).
- `ST77916_init_V1.0.INI` - oficjalna sekwencja inicjalizacji ekranu.
- `JC3636K518_Instructions-EN.pdf` - instrukcja producenta.

> Status: **niezweryfikowane na żywo.** Nie mamy płytki K518 do testów; piny
> pochodzą ze schematu, ale audio przez CH445P i kierunek knoba wymagają
> potwierdzenia na sprzęcie.
