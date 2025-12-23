# ESP32-S3-Touch-LCD-1.85C - Technical Solution

Technical documentation on getting the Waveshare ESP32-S3-Touch-LCD-1.85**C** variant working with ESPHome.

> 📖 For setup instructions, see [README.md](README.md)

---

## 🔴 The Problem

When using ESPHome configurations from the Home Assistant forum for the **ESP32-S3-Touch-LCD-1.85C** (note the "C" variant), the display showed:
- "Snow" effect - colored horizontal lines/static
- No proper display output despite device connecting to WiFi

This is because the **C variant requires a completely different initialization sequence** than the non-C variant discussed in forum posts.

---

## 🔍 Root Cause Analysis

### What Didn't Work

Forum configurations used:
- Reset pin 0 or 1 on PCA9554
- 40MHz data rate
- Generic ST77916 init sequences
- No color inversion

### The Discovery

The official Waveshare demo code (downloaded from their wiki) contains a **different** init sequence called `vendor_specific_init_new[]` in:

```
ESP32-S3-Touch-LCD-1.85-Demo/ESP-IDF/main/LCD_Driver/ST77916.c
```

This init sequence is specifically for the **C variant** and is completely different from what's shared in forum posts.

---

## ✅ The Solution

### Critical Settings for 1.85C

| Setting | Correct Value | Incorrect Value |
|---------|---------------|-----------------|
| Reset Pin | PCA9554 Pin **2** | Pin 0 or 1 |
| Data Rate | **80MHz** | 40MHz |
| Color Inversion | **true** | false |
| Init Sequence | Waveshare official | Forum gists |

### ESPHome Configuration

```yaml
display:
  - platform: qspi_dbi
    model: CUSTOM
    data_rate: 80MHz              # Higher than typical 40MHz
    color_order: rgb
    invert_colors: true           # Required for correct colors
    dimensions:
      height: 360
      width: 360
    cs_pin: GPIO21
    reset_pin:
      pca9554: pca9554_device
      number: 2                   # EXIO2 - Critical for 1.85C!
    auto_clear_enabled: false     # Required for LVGL
    update_interval: never        # LVGL handles updates
    init_sequence:
      # ... Official Waveshare init sequence ...
```

### Color Inversion Issue

Without `invert_colors: true`:
- Sending GREEN showed **PURPLE**
- Sending RED showed **CYAN**

The fix requires **both**:
1. ESPHome `invert_colors: true` setting
2. Display inversion command `0x21` in init sequence

---

## 🔧 Hardware Differences

### C Variant Specifics

| Component | 1.85C Variant | Non-C Variant |
|-----------|---------------|---------------|
| Display Resolution | 360×360 | 360×360 |
| Display Controller | ST77916 | ST77916 |
| **Reset Pin** | **PCA9554 Pin 2** | PCA9554 Pin 0/1 |
| **Data Rate** | **80MHz** | 40MHz |
| **Init Sequence** | vendor_specific_init_new[] | Different |
| I2C Bus | GPIO10/11 | GPIO10/11 |

### I2C Device Map

```
Address 0x15 - CST816 Touch Controller
Address 0x20 - PCA9554 I/O Expander  
Address 0x51 - EEPROM (unused by ESPHome)
```

### Button Behavior

| Button | Location | GPIO | Behavior |
|--------|----------|------|----------|
| Top | Right side | GPIO0 | Programmable (also boot button) |
| Bottom | Right side | - | Hardware reset (EN pin) |
| Slide | Left side | - | Hardware power switch |

> ⚠️ GPIO0 is the ESP32 boot button - pressing it may cause resets during boot. This is normal.

---

## 📋 Complete Init Sequence

The working init sequence from official Waveshare code:

```yaml
init_sequence:
  # Vendor-specific initialization
  - [0xF0, 0x28]
  - [0xF2, 0x28]
  - [0x73, 0xF0]
  - [0x7C, 0xD1]
  - [0x83, 0xE0]
  - [0x84, 0x61]
  - [0xF2, 0x82]
  - [0xF0, 0x00]
  - [0xF0, 0x01]
  - [0xF1, 0x01]
  - [0xB0, 0x56]
  - [0xB1, 0x4D]
  - [0xB2, 0x24]
  # ... (see full YAML for complete sequence) ...
  
  # Final commands
  - [0x3A, 0x55]              # Color mode RGB565
  - [0x36, 0x00]              # MADCTL - RGB order
  - [0x2A, 0x00, 0x00, 0x01, 0x67]  # Column address (0-359)
  - [0x2B, 0x00, 0x00, 0x01, 0x67]  # Row address (0-359)
  - [0x21, 0x00]              # Display inversion ON
  - [0x11, 0x00]              # Sleep out
  - delay 120ms
  - [0x29, 0x00]              # Display ON
  - delay 20ms
```

---

## 🧪 Debugging Steps Used

### 1. Initial Investigation

```bash
# Checked I2C devices
[I][i2c.arduino:096]: Found device at address 0x15 (CST816)
[I][i2c.arduino:096]: Found device at address 0x20 (PCA9554)
[I][i2c.arduino:096]: Found device at address 0x51 (EEPROM)
```

### 2. Pin Testing

Tested reset pin combinations:
- Pin 0: Display stayed black
- Pin 1: Display showed snow
- **Pin 2: Display worked!**

### 3. Data Rate Testing

- 40MHz: Snow/artifacts
- **80MHz: Clean display**

### 4. Color Testing

Created test config with solid color fill:
```yaml
lambda: |-
  it.fill(Color(0, 255, 0));  // GREEN
```

Result: Displayed purple → Added `invert_colors: true` → Correct green

---

## 💡 Key Takeaways

1. **The "C" suffix matters** - It's a different hardware revision requiring different configuration.

2. **Forum configs may not work** - Always check for official vendor code when troubleshooting.

3. **Reset pin is critical** - The C variant specifically uses PCA9554 pin 2 (EXIO2).

4. **Data rate matters** - 80MHz is required for this variant (higher than typical 40MHz).

5. **Color inversion is double** - Both init command (0x21) AND ESPHome `invert_colors: true` are needed.

6. **PSRAM configuration** - ESP-IDF framework required with specific sdkconfig_options.

---

## 📚 Reference Materials

### Official Resources

- **Waveshare Wiki**: https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.85
- **Official Demo Download**: https://files.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.85/ESP32-S3-Touch-LCD-1.85-Demo.zip

### Source File Location (in demo zip)

```
ESP32-S3-Touch-LCD-1.85-Demo/
└── ESP-IDF/
    └── main/
        └── LCD_Driver/
            └── ST77916.c    # Contains vendor_specific_init_new[]
```

### Community Resources

- **Home Assistant Forum**: https://community.home-assistant.io/t/waveshare-esp32-s3-lcd-1-85/833702
- **ESPHome QSPI DBI**: https://esphome.io/components/display/qspi_dbi.html

---

## ✅ Final Status

| Feature | Status |
|---------|--------|
| Display | ✅ Working |
| LVGL UI | ✅ Working (3 pages) |
| Touch | ✅ Working |
| Auto-sleep | ✅ Working |
| WiFi | ✅ Working |
| Button | ✅ Cycles pages |
| Voice Assistant | ✅ Ready (needs HA pipeline) |
| Microphone | ✅ Configured |
| Speaker | ✅ Configured |

---

*Display working! Configuration complete for the 1.85C variant.* 🎉
