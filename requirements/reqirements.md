Ein sauberes Minimalprojekt aufzusetzen. Für das **STM32H750B-DK** gibt es sogar eine passende Display-Anwendung im STM32CubeH7-Paket: **`Projects/STM32H750B-DK/Applications/Display/LTDC_Paint`**. Zusätzlich liefert der offizielle BSP-Treiber genau die LTDC- und SDRAM-Initialisierung für das Board. ([GitHub](https://github.com/STMicroelectronics/STM32CubeH7/blob/master/Projects/STM32H750B-DK/Applications/Display/LTDC_Paint/readme.txt))

## 1) Die zwei wichtigsten offiziellen Quellen

**Board-BSP für H750B-DK**
Darin stehen die Board-spezifischen Initialisierungen für LCD, LTDC, Touch und SDRAM. Der LCD-Treiber sagt ausdrücklich, dass er das TFT **direkt über LTDC** ansteuert und die **Timings des RK043FN48H** verwendet. Der SDRAM-Treiber ist für das auf dem Board verbaute **MT48LC4M32B2B5-6A**. ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))

**STM32CubeH7 Beispiel `LTDC_Paint`**
Das Beispiel läuft auf dem **STM32H750B-DK**, beschreibt die LCD-/Touch-Konfiguration und ist ein sehr guter Referenzpunkt, auch wenn es kein „nacktes Hello-Framebuffer“ ist. ([GitHub](https://github.com/STMicroelectronics/STM32CubeH7/blob/master/Projects/STM32H750B-DK/Applications/Display/LTDC_Paint/readme.txt))

## 2) Was du in CubeMX wirklich einstellen solltest

Für dieses Board **nicht** `X-CUBE-DISPLAY -> LCD -> ILI9341/ST7789` verwenden. Das RK043FN48H ist hier kein SPI-Display mit eigenem Controller, sondern ein **RGB-Panel**, das über **LTDC + SDRAM** betrieben wird. Der ST-BSP initialisiert dafür LTDC, Layer und SDRAM automatisch über `BSP_LCD_Init()` bzw. `BSP_LCD_InitEx()`. ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))

In CubeMX für ein Minimalprojekt:

### Board / Grundsetup

- Board: **STM32H750B-DK**
- CPU z. B. 400 MHz wie im ST-Beispiel `LTDC_Paint` ([GitHub](https://github.com/STMicroelectronics/STM32CubeH7/blob/master/Projects/STM32H750B-DK/Applications/Display/LTDC_Paint/readme.txt))

### Peripherien aktivieren

- **FMC** für SDRAM
- **LTDC**
- optional **DMA2D** für spätere Beschleunigung
- optional **I2C4** nur dann, wenn du Touch brauchst; für das erste Bild nicht zwingend

### Display-Middleware

- **nichts** unter `X-CUBE-DISPLAY` auswählen
- TouchGFX ist optional, für dein Ziel „ich kontrolliere die Pipeline selbst“ würde ich es zunächst **weglassen**

## 3) Exakte LTDC-Werte für das H750B-DK

Die offiziellen Boardwerte stehen direkt im BSP. Der BSP setzt:

- HSPolarity = `AL`
- VSPolarity = `AL`
- DEPolarity = `AL`
- PCPolarity = `IPC` ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))

Die Clock-Konfiguration für das LCD ist im BSP ebenfalls fest vorgegeben:

- PLL3M = 5
- PLL3N = 160
- PLL3R = 83
- LTDC clock ≈ **9.63 MHz** ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))

Die LTDC-Timingformeln im BSP basieren auf den RK043FN48H-Makros und werden so gesetzt:
`HorizontalSync = RK043FN48H_HSYNC - 1`
`AccumulatedHBP = RK043FN48H_HSYNC + (RK043FN48H_HBP - 11) - 1`
`AccumulatedActiveW = RK043FN48H_HSYNC + Width + RK043FN48H_HBP - 1`
`TotalWidth = RK043FN48H_HSYNC + Width + (RK043FN48H_HBP - 11) + RK043FN48H_HFP - 1`
`VerticalSync = RK043FN48H_VSYNC - 1`
`AccumulatedVBP = RK043FN48H_VSYNC + RK043FN48H_VBP - 1`
`AccumulatedActiveH = RK043FN48H_VSYNC + Height + RK043FN48H_VBP - 1`
`TotalHeigh = RK043FN48H_VSYNC + Height + RK043FN48H_VBP + RK043FN48H_VFP - 1` ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))

Wenn du exakt board-kompatibel bleiben willst, ist der beste Weg: **diese Funktionen aus dem BSP übernehmen oder direkt verwenden**, statt die Timingzahlen von Hand neu zu erraten. ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))

## 4) Exakte SDRAM-Werte für das H750B-DK

Der offizielle SDRAM-BSP setzt für das Board:

- Bank = `FMC_SDRAM_BANK2`
- Column bits = 8
- Row bits = 12
- Bus width = 16 Bit
- Internal banks = 4
- CAS latency = 3
- SDClockPeriod = 2
- ReadBurst = Enable
- ReadPipeDelay = 0 ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk/blob/main/stm32h750b_discovery_sdram.c))

Timing:

- LoadToActiveDelay = 2
- ExitSelfRefreshDelay = 7
- SelfRefreshTime = 4
- RowCycleDelay = 7
- WriteRecoveryTime = 2
- RPDelay = 2
- RCDDelay = 2 ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk/blob/main/stm32h750b_discovery_sdram.c))

Der BSP initialisiert das SDRAM vor der LTDC-Layer-Konfiguration ausdrücklich mit `BSP_SDRAM_Init(0)`. ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))

## 5) Framebuffer-Adresse / Linker-Script

Der Board-Konfigurationstemplate von ST definiert:

- `LCD_LAYER_0_ADDRESS = 0xD0000000`
- `LCD_LAYER_1_ADDRESS = 0xD0200000` ([GitHub](https://raw.githubusercontent.com/STMicroelectronics/stm32h750b-dk-bsp/main/stm32h750b_discovery_conf_template.h))

Für ein Minimalprojekt mit **einem Layer** reicht also:

```ld
MEMORY
{
  FLASH    (rx)  : ORIGIN = 0x08000000, LENGTH = 128K
  RAM_D1   (xrw) : ORIGIN = 0x24000000, LENGTH = 512K
  SDRAM    (xrw) : ORIGIN = 0xD0000000, LENGTH = 8M
}
```

Und für einen dedizierten Framebuffer-Abschnitt:

```ld
.framebuffer (NOLOAD) :
{
  . = ALIGN(32);
  *(.framebuffer)
  . = ALIGN(32);
} > SDRAM
```

Die 32-Byte-Ausrichtung ist sinnvoll wegen Cache-/DMA-Kohärenz; ST weist im `LTDC_Paint`-Readme ausdrücklich auf Cache-Kohärenz und 32-Byte-Alignment für gemeinsam genutzte Buffer hin. ([GitHub](https://github.com/STMicroelectronics/STM32CubeH7/blob/master/Projects/STM32H750B-DK/Applications/Display/LTDC_Paint/readme.txt))

## 6) Minimaler C-Code: erstes Bild sichtbar

Der schnellste und robusteste Weg ist, **den BSP zu benutzen** und den Framebuffer dann selbst zu füllen:

```c
#include "stm32h750b_discovery.h"
#include "stm32h750b_discovery_lcd.h"
#include "stm32h750b_discovery_sdram.h"

#define FB_ADDR ((uint32_t *)0xD0000000U)
#define LCD_W   480U
#define LCD_H   272U

static void FillScreen(uint32_t color)
{
    for (uint32_t y = 0; y < LCD_H; ++y)
    {
        for (uint32_t x = 0; x < LCD_W; ++x)
        {
            FB_ADDR[y * LCD_W + x] = color;   // ARGB8888
        }
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();   // deine normale H750 Clock-Init

    if (BSP_LCD_Init(0, LCD_ORIENTATION_LANDSCAPE) != BSP_ERROR_NONE)
    {
        Error_Handler();
    }

    BSP_LCD_DisplayOn(0);

    FillScreen(0xFFFF0000U);   // rot
    HAL_Delay(1000);
    FillScreen(0xFF00FF00U);   // grün
    HAL_Delay(1000);
    FillScreen(0xFF0000FFU);   // blau

    while (1)
    {
    }
}
```

Das passt zur offiziellen BSP-Logik: `BSP_LCD_Init()` initialisiert LTDC, LTDC-Clock, Layer und — falls nötig — auch das SDRAM; der Default-Layer wird mit `LCD_LAYER_0_ADDRESS` konfiguriert. ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))

## 7) CubeMX: konkrete Reihenfolge

Wenn du das in CubeMX nachbaust, würde ich so vorgehen:

1. Neues Projekt für **STM32H750B-DK** anlegen.
2. **FMC** aktivieren und die SDRAM-Parameter auf die ST-BSP-Werte setzen. ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk/blob/main/stm32h750b_discovery_sdram.c))
3. **LTDC** aktivieren.
4. LTDC-Clock auf **PLL3** legen und die BSP-Werte `M=5, N=160, R=83` übernehmen. ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))
5. Layer 0 auf **ARGB8888** setzen, Startadresse **`0xD0000000`**. ([GitHub](https://raw.githubusercontent.com/STMicroelectronics/stm32h750b-dk-bsp/main/stm32h750b_discovery_conf_template.h))
6. `X-CUBE-DISPLAY` auf **Not selected** lassen.
7. Den generierten Code mit den **BSP-Dateien** ergänzen und zuerst `BSP_LCD_Init()` als Referenz nutzen.

## 8) Warum ich dir den BSP-Weg empfehle

Der ST-BSP ist hier die sicherste Wahrheit für das Board, weil er genau die Kombination aus

- RK043FN48H,
- H750B-DK Pinbelegung,
- LTDC-Clock,
- LTDC-Timing,
- SDRAM-Initialisierung

bereits zusammenführt. Außerdem gibt es im CubeH7-Paket mit `LTDC_Paint` ein offizielles H750B-DK-Displayprojekt als Referenz. ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))

## 9) Wo du direkt schauen solltest

- H750B-DK LCD BSP: `stm32h750b_discovery_lcd.c` ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk-bsp/blob/main/stm32h750b_discovery_lcd.c))
- H750B-DK SDRAM BSP: `stm32h750b_discovery_sdram.c` ([GitHub](https://github.com/STMicroelectronics/stm32h750b-dk/blob/main/stm32h750b_discovery_sdram.c?utm_source=chatgpt.com))
- H750B-DK Config-Template mit Layer-Adressen: `stm32h750b_discovery_conf_template.h` ([GitHub](https://raw.githubusercontent.com/STMicroelectronics/stm32h750b-dk-bsp/main/stm32h750b_discovery_conf_template.h))
- CubeH7 Beispiel: `Projects/STM32H750B-DK/Applications/Display/LTDC_Paint` ([GitHub](https://github.com/STMicroelectronics/STM32CubeH7/blob/master/Projects/STM32H750B-DK/Applications/Display/LTDC_Paint/readme.txt))

Im aktuellen STM32CubeH7-Release gibt es weiterhin Boardprojekte, aber einige ältere Demonstrationen und STemWin-Anwendungen wurden entfernt; das erklärt, warum man nicht immer ein ganz kleines „blank screen“-Beispiel findet. ([GitHub](https://github.com/STMicroelectronics/STM32CubeH7/blob/master/Release_Notes.html))