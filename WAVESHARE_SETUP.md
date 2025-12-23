# Waveshare ESP32-S3 LCD 1.85" Quick Reference

Quick reference guide for setting up and configuring your Waveshare ESP32-S3-Touch-LCD-1.85C.

> 📖 For complete documentation, see [README.md](README.md)

## 🔌 USB Setup (Linux)

```bash
# Watch for device connection
sudo dmesg -w

# Grant access to USB port
sudo chmod 666 /dev/ttyACM0

# Or add user to dialout group (permanent fix)
sudo usermod -a -G dialout $USER
# Log out and back in after this
```

## 📋 Secrets Template

Create `secret.yaml` with:

```yaml
wifi_ssid: "YourWiFiName"
wifi_password: "YourWiFiPassword"
api_encryption_key: "generate-a-32-char-key-here!!!!"
ota_password: "your-ota-password"
ap_password: "fallback-ap-password"
```

Generate API key:
```bash
openssl rand -base64 32
```

## 🎯 Pin Reference

### Display (QSPI)
| Pin | GPIO |
|-----|------|
| CLK | GPIO40 |
| D0 | GPIO46 |
| D1 | GPIO45 |
| D2 | GPIO42 |
| D3 | GPIO41 |
| CS | GPIO21 |
| Reset | PCA9554 Pin **2** |
| Backlight | GPIO5 |

### I2C Bus
| Pin | GPIO |
|-----|------|
| SDA | GPIO11 |
| SCL | GPIO10 |

### I2C Devices
| Address | Device |
|---------|--------|
| 0x15 | CST816 Touch |
| 0x20 | PCA9554 I/O Expander |
| 0x51 | EEPROM (unused) |

### Touch
| Pin | GPIO |
|-----|------|
| Interrupt | GPIO4 |

### Audio
| Function | GPIO |
|----------|------|
| Mic LRCLK | GPIO2 |
| Mic BCLK | GPIO15 |
| Mic DIN | GPIO39 |
| Speaker LRCLK | GPIO38 |
| Speaker BCLK | GPIO48 |
| Speaker DOUT | GPIO47 |

### Buttons
| Button | GPIO | Note |
|--------|------|------|
| Side Button | GPIO0 | Also boot button |
| Slide Switch | N/A | Hardware only |

## ⚡ Flashing Commands

```bash
# First flash (USB)
esphome run waveshare-esp32-s3-lcd-185.yaml

# OTA update
esphome run waveshare-esp32-s3-lcd-185.yaml --device esp32-s3-touch-lcd-185.local

# Compile only
esphome compile waveshare-esp32-s3-lcd-185.yaml

# View logs
esphome logs waveshare-esp32-s3-lcd-185.yaml
```

## 🔧 Critical Settings for 1.85C

These settings are **required** for the C variant:

```yaml
# Display data rate
data_rate: 80MHz

# Reset pin on PCA9554
reset_pin:
  pca9554: pca9554_device
  number: 2  # NOT 0 or 1!

# Color inversion fix
invert_colors: true

# PSRAM mode
psram:
  mode: octal
  speed: 80MHz
```

## ❌ Common Errors

| Error | Fix |
|-------|-----|
| Display shows "snow" | Check reset pin is 2, data rate is 80MHz |
| Colors inverted | Add `invert_colors: true` |
| PSRAM error | Check sdkconfig_options in esp32 section |
| I2C not found | Verify SDA=GPIO11, SCL=GPIO10 |
| Touch not working | Check interrupt pin GPIO4 |
| USB permission denied | `sudo chmod 666 /dev/ttyACM0` |

## 🔗 Quick Links

- [README.md](README.md) - Full documentation
- [VOICE_ASSISTANT_SETUP.md](VOICE_ASSISTANT_SETUP.md) - Voice assistant guide
- [ESP32-S3-TOUCH-LCD-185C-SOLUTION.md](ESP32-S3-TOUCH-LCD-185C-SOLUTION.md) - Technical details
- [Waveshare Wiki](https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.85)
- [HA Forum Thread](https://community.home-assistant.io/t/waveshare-esp32-s3-lcd-1-85/833702)
