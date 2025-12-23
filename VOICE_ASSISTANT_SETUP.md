# Voice Assistant Setup Guide

Complete guide to setting up voice assistant on the Waveshare ESP32-S3-Touch-LCD-1.85C with Home Assistant.

## ✅ What Works

| Feature | Status |
|---------|--------|
| **Push-to-talk** | ✅ Press button, speak, release |
| **Wake word** | ✅ "Hey Jarvis" (requires openWakeWord) |
| **Speech-to-text** | ✅ Via Whisper |
| **Text-to-speech** | ✅ Via Piper |
| **Display feedback** | ✅ Shows Listening/Processing/Ready/Error |

## 🚀 Quick Setup

### Prerequisites

- Home Assistant with ESPHome integration
- ESP32-S3-Touch-LCD-1.85C flashed with this config
- Local or cloud voice processing (Whisper + Piper recommended)

---

## Option A: Local Voice Processing (Recommended)

### Step 1: Install Add-ons

In Home Assistant, go to **Settings → Add-ons → Add-on Store**:

1. **Whisper** (Speech-to-text)
   - Search for "Whisper"
   - Click Install
   - Configuration:
     ```yaml
     language: en
     model: base-int8  # Or tiny-int8 for faster/lower accuracy
     beam_size: 1
     ```
   - Enable "Start on boot" and "Watchdog"
   - Click Start

2. **Piper** (Text-to-speech)
   - Search for "Piper"
   - Click Install
   - Configuration:
     ```yaml
     voice: en_US-lessac-medium
     ```
   - Enable "Start on boot" and "Watchdog"
   - Click Start

3. **openWakeWord** (Optional - for wake word detection)
   - Search for "openWakeWord"
   - Click Install
   - Enable "Start on boot" and "Watchdog"
   - Click Start

### Step 2: Add Wyoming Integrations

Go to **Settings → Devices & Services → + Add Integration**:

1. Search for **Wyoming Protocol**
2. Add for Whisper:
   - Host: `core-whisper`
   - Port: `10300`
3. Add for Piper:
   - Host: `core-piper`
   - Port: `10200`
4. Add for openWakeWord (if installed):
   - Host: `core-openwakeword`
   - Port: `10400`

### Step 3: Create Voice Pipeline

1. Go to **Settings → Voice assistants**
2. Click **+ Add Assistant**
3. Configure:
   - **Name**: `Home Assistant` (or your choice)
   - **Language**: `English`
   - **Conversation agent**: `Home Assistant`
   - **Speech-to-text**: `faster-whisper`
   - **Text-to-speech**: `piper`
   - **Wake word**: `openwakeword` (if installed)
4. Click **Create**

### Step 4: Assign Pipeline to Device

1. Go to **Settings → Devices & Services → ESPHome**
2. Click on **ESP32-S3 Touch LCD 1.85**
3. Click **Configure**
4. Set **Assistant** to your pipeline
5. Set **Wake word** to `hey_jarvis` (or your preference)
6. Click **Submit**

---

## Option B: Docker-based Voice Processing

For better performance, run Whisper/Piper in Docker with GPU acceleration.

### docker-compose.yml

```yaml
version: '3'
services:
  whisper:
    container_name: whisper
    image: rhasspy/wyoming-whisper:latest
    restart: unless-stopped
    command: --model base-int8 --language en
    volumes:
      - ./whisper-data:/data
    ports:
      - "10300:10300"
    # For AMD GPU acceleration:
    devices:
      - /dev/dri:/dev/dri
    group_add:
      - "44"    # video group
      - "993"   # render group

  piper:
    container_name: piper
    image: rhasspy/wyoming-piper:latest
    restart: unless-stopped
    command: --voice en_US-lessac-medium
    volumes:
      - ./piper-data:/data
    ports:
      - "10200:10200"

  openwakeword:
    container_name: openwakeword
    image: rhasspy/wyoming-openwakeword:latest
    restart: unless-stopped
    command: --preload-model 'hey_jarvis'
    ports:
      - "10400:10400"
```

### Start Services

```bash
docker-compose up -d
```

### Configure Wyoming in Home Assistant

Add Wyoming integrations pointing to your Docker host:

1. **Settings → Devices & Services → + Add Integration**
2. Search for **Wyoming Protocol**
3. Add each service:
   - Whisper: `<docker-host-ip>:10300`
   - Piper: `<docker-host-ip>:10200`
   - openWakeWord: `<docker-host-ip>:10400`

---

## 🎤 Using Voice Assistant

### Push-to-Talk (Most Reliable)

1. **Press** the side button on the device
2. Display wakes and shows "Ready"
3. **Speak** your command clearly
4. Display shows "Listening..."
5. **Wait** for response
6. Display shows "Processing..." then plays audio response

### Wake Word

1. Say **"Hey Jarvis"** (or your configured wake word)
2. Display wakes and shows "Listening..."
3. **Speak** your command
4. Wait for response

### Example Commands

- "What time is it?"
- "Turn on the living room lights"
- "What's the weather like?"
- "Set a timer for 5 minutes"
- "Add milk to my shopping list"

---

## 🔧 Whisper Model Comparison

| Model | Speed | Accuracy | RAM | Best For |
|-------|-------|----------|-----|----------|
| `tiny-int8` | ⚡⚡⚡ | 😐 OK | ~200MB | Low-power hardware |
| `base-int8` | ⚡⚡ | 👍 Good | ~400MB | Recommended balance |
| `small-int8` | ⚡ | 👍👍 Better | ~1GB | Better accuracy |
| `medium` | 🐢 | 👍👍👍 Great | ~3GB | High accuracy |

**Recommendation**: Start with `base-int8`. Use `tiny-int8` on weak hardware, `small-int8` for better accuracy.

---

## 🐛 Troubleshooting

### Voice assistant doesn't respond

1. **Check pipeline is assigned**: Settings → Devices → ESPHome → Your device → Configure
2. **Check add-ons are running**: Settings → Add-ons → Whisper/Piper status
3. **Check Wyoming integrations**: Settings → Devices & Services → Wyoming Protocol
4. **Check device logs**: ESPHome web interface or HA logs

### "Listening" but no response

- Whisper may not be receiving audio
- Check Whisper add-on logs for "Processing audio..." messages
- Try speaking louder/clearer
- Check microphone pins are correct

### Wake word not working

1. **Check openWakeWord is running**: Settings → Add-ons → openWakeWord
2. **Check Wyoming integration**: Must be connected for openWakeWord
3. **Select wake word on device**: Settings → Devices → ESPHome → Your device → Wake word dropdown
4. **Low-power hardware issue**: Wake word requires constant processing; may not work reliably on thin clients
5. **Use push-to-talk as fallback**: More reliable on all hardware

### Slow responses

- Use `tiny-int8` or `base-int8` Whisper model
- Consider GPU-accelerated Docker setup
- Upgrade Home Assistant hardware
- Check network latency between HA and device

### "Connection lost" errors

- Common on low-powered hardware (thin clients)
- Whisper/Piper/openWakeWord competing for resources
- Solutions:
  - Disable openWakeWord (use push-to-talk)
  - Use faster Whisper model (`tiny-int8`)
  - Upgrade to more powerful hardware
  - Use Docker with GPU acceleration

---

## 📊 Hardware Recommendations

| Setup | Whisper | Wake Word | Notes |
|-------|---------|-----------|-------|
| **Thin client** | `tiny-int8` | ❌ Disable | Push-to-talk only |
| **Raspberry Pi 4** | `base-int8` | ⚠️ Maybe | May be slow |
| **Intel NUC / Mini PC** | `base-int8` | ✅ Works | Good balance |
| **PC with GPU** | `small-int8` or larger | ✅ Works | Best experience |

---

## 🎯 Best Practices

1. **Start simple**: Get push-to-talk working first
2. **Add wake word later**: Once push-to-talk is reliable
3. **Match hardware to features**: Don't expect everything on a thin client
4. **Use GPU acceleration**: Dramatically improves response time
5. **Monitor resources**: Check CPU/RAM usage during voice processing

---

## 📚 Resources

- [ESPHome Voice Assistant](https://esphome.io/components/voice_assistant.html)
- [Home Assistant Voice](https://www.home-assistant.io/voice_control/)
- [Wyoming Protocol](https://github.com/rhasspy/wyoming)
- [Whisper Models](https://github.com/openai/whisper#available-models-and-languages)
- [Piper Voices](https://rhasspy.github.io/piper-samples/)

---

**Enjoy your voice-enabled smart display!** 🎤🖥️
